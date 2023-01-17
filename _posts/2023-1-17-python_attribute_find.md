---
layout: post
title: Python属性查找
date: 2023-1-17 17:18:23 +0800
categories: Python
tags: 属性查找 源码分析 
author: Rxsi
---

* content
{:toc}


# 两个问题
今天在公司有同事提出了一个问题：
```python
class A:
    def FuncA(self):
        pass

a = A()
a.FuncA = a.FuncA
```
以上代码是否会造成内存泄漏问题？
另外根据这个问题涉及到的相关特性，再思考下这个问题：
```python
class Descr:
    def __init__(self):
        self._instance = None
        self._val = 0

    def __get__(self, instance, owner):
        print("descr.__get__")
        self._instance = instance
        return self._val

    def __set__(self, instance, value):
        print("descr.__set__")
        self._instance = instance
        self._val = value

class A:
    obj = Descr()

a = A()
a.obj = a.obj
```
以上代码是否会造成内存泄漏？如果把`__set__`去掉呢？
# 属性查找
要解答上面的两个问题，我们需要先了解 Python 是如何进行属性查找的。
我们知道`a.FuncA`实际等价于`getattr(a, "FuncA")`，通过 getattr 函数，我们从源码入手查看底层的实现奥秘。要搜索某个内置函数的源码实现，一种技巧就是直接搜索对应内置函数名的字符串形式，通过搜索我们可以定位到代码如下：
```c
static PyMethodDef builtin_methods[] = {
    // ...
    {"getattr",         builtin_getattr,    METH_VARARGS, getattr_doc},
    // ...
};
```
再根据目标函数跳转，最后我们可以定位到最终实际实现函数：
```c
/* Generic GetAttr functions - put these in your tp_[gs]etattro slot */

PyObject *
_PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict) // obj是实例对象，name是目标属性名，dict初始默认是NULL
{
    PyTypeObject *tp = Py_TYPE(obj); // 获取实例对象的PyTypeObject对象
    PyObject *descr = NULL;
    PyObject *res = NULL;
    descrgetfunc f;
    Py_ssize_t dictoffset;
    PyObject **dictptr;

    if (!PyUnicode_Check(name)){ // 检查属性名是否合法
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     name->ob_type->tp_name);
        return NULL;
    }
    Py_INCREF(name);

    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            goto done;
    }

    descr = _PyType_Lookup(tp, name); // 根据mro顺序逐层查找目标属性

    f = NULL;
    if (descr != NULL) { // 说明该属性存在于某个类的__dict__中
        Py_INCREF(descr);
        f = descr->ob_type->tp_descr_get; // 获取目标属性的tp_descr_get函数，即__get__魔法函数
        if (f != NULL && PyDescr_IsData(descr)) { // 如果目标属性同时实现了__set__魔法函数，证明是数据描述符
            res = f(descr, obj, (PyObject *)obj->ob_type); // 1）调用数据描述符获取结果
            goto done;
        }
    }
	// 非数据描述符
    if (dict == NULL) { // 获取实例对象的__dict__属性
        /* Inline _PyObject_GetDictPtr */
        dictoffset = tp->tp_dictoffset;
        if (dictoffset != 0) {
            if (dictoffset < 0) {
                Py_ssize_t tsize;
                size_t size;

                tsize = ((PyVarObject *)obj)->ob_size;
                if (tsize < 0)
                    tsize = -tsize;
                size = _PyObject_VAR_SIZE(tp, tsize);
                assert(size <= PY_SSIZE_T_MAX);

                dictoffset += (Py_ssize_t)size;
                assert(dictoffset > 0);
                assert(dictoffset % SIZEOF_VOID_P == 0);
            }
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        }
    }
    if (dict != NULL) {
        Py_INCREF(dict);
        res = PyDict_GetItem(dict, name); // 2）从实例对象的__dict__中查找目标属性
        if (res != NULL) {
            Py_INCREF(res);
            Py_DECREF(dict);
            goto done;
        }
        Py_DECREF(dict);
    }

    if (f != NULL) { // 3）只实现了__get__函数
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        goto done;
    }

    if (descr != NULL) { // 4）属于类变量
        res = descr;
        descr = NULL;
        goto done;
    }

    PyErr_Format(PyExc_AttributeError,
                 "'%.50s' object has no attribute '%U'",
                 tp->tp_name, name);
  done:
    Py_XDECREF(descr);
    Py_DECREF(name);
    return res;
}
```
从上面的源码注释可以知道，实际查找顺序为：

1. 数据描述符，即目标属性属于类属性且实现了`__get__`和`__set__`魔法函数；
2. 实例对象的变量；
3. 非数据描述符，即只实现了`__get__`魔法函数；
4. 类变量；

如果以上都查找不到，那么抛出异常
## a.FuncA的查找顺序
那么当我们调用`a.FuncA`时是从哪一层获取到目标属性的呢？
我们知道 FuncA 是一个定义在类 A 中的函数，在代码运行时会生成一个函数对象，且存放于在类 A 的 __dict__ 中：
```python
>>> print(A.__dict__)
{'FuncA': <function A.FuncA at 0x00000254011C9AE8>} # 省略部分无用输出
```
那么根据上面源码部分可知，将会根据函数对象是否实现了`__get__`和`__set__`魔法函数而进入不同的分支
查看函数对象对应的 PyTypeObject 实现：
```python
PyTypeObject PyFunction_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "function",
    sizeof(PyFunctionObject),
    0,
    (destructor)func_dealloc,                   /* tp_dealloc */
    0,                                          /* tp_print */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_reserved */
    (reprfunc)func_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    function_call,                              /* tp_call */
    0,                                          /* tp_str */
    0,                                          /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC,/* tp_flags */
    func_doc,                                   /* tp_doc */
    (traverseproc)func_traverse,                /* tp_traverse */
    0,                                          /* tp_clear */
    0,                                          /* tp_richcompare */
    offsetof(PyFunctionObject, func_weakreflist), /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    0,                                          /* tp_methods */
    func_memberlist,                            /* tp_members */
    func_getsetlist,                            /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    func_descr_get,                             /* tp_descr_get */ // __get__ 魔法函数
    0,                                          /* tp_descr_set */ // __set__ 魔法函数，没有实现
    offsetof(PyFunctionObject, func_dict),      /* tp_dictoffset */
    0,                                          /* tp_init */
    0,                                          /* tp_alloc */
    func_new,                                   /* tp_new */
};

```
从 PyFunction_Type 的实现可以知道，函数对象只实现了`__get__`魔法函数，因此最终的查找顺序为：

1. 实例对象的变量；
2. 非数据描述符；

实例对象 a 的名字空间很明显是不存在 FuncA 属性的，因此查找顺序最后就定位到了非数据描述符的查找了，在上面的源码中，调用方式为：
```python
f = descr->ob_type->tp_descr_get;
// ...
res = f(descr, obj, (PyObject *)Py_TYPE(obj));
```
实际调用的函数就是 PyFunction_Type.func_descr_get，查看具体实现：
```c
/* Bind a function to an object */
static PyObject *
func_descr_get(PyObject *func, PyObject *obj, PyObject *type)
  {
      if (obj == Py_None || obj == NULL) {
      Py_INCREF(func);
      return func;
  }
      return PyMethod_New(func, obj);
  }


/* Method objects are used for bound instance methods returned by
   instancename.methodname. ClassName.methodname returns an ordinary
   function.
*/

PyObject *
PyMethod_New(PyObject *func, PyObject *self) // func函数对象，self是实例对象
{
    PyMethodObject *im; // Method方法对象
    if (self == NULL) {
        PyErr_BadInternalCall();
        return NULL;
    }
    im = free_list;
    if (im != NULL) {
        free_list = (PyMethodObject *)(im->im_self);
        (void)PyObject_INIT(im, &PyMethod_Type);
        numfree--;
    }
    else {
        im = PyObject_GC_New(PyMethodObject, &PyMethod_Type);
        if (im == NULL)
            return NULL;
    }
    im->im_weakreflist = NULL;
    Py_INCREF(func);
    im->im_func = func;
    Py_XINCREF(self);
    im->im_self = self; // 这里引用了实例对象！！
    _PyObject_GC_TRACK(im);
    return (PyObject *)im;
}
```
可以看到最终返回的是一个 **新创建 **的方法对象，称为`bound method`，如以下输出：
```python
>>> print(a.FuncA)
<bound method A.FuncA of <__main__.A object at 0x0000018D417A89B0>>

# 如果直接从类的属性字典中获取得到的则是函数对象
>>> print(A.FuncA)
<function A.FuncA at 0x0000018764629A60>
```
同时这个方法对象在内部保存了实例对象的引用，因为这个方法对象需要把实例对象（即 self 参数）作为首个函数参数进行调用。
至此，我们再回看问题 1 的定义方式：
```c
a.FuncA = a.FuncA
```
后面`a.FuncA`得到的是一个 bound Method，这个对象保存了 a 的引用，而前面的`a.FuncA`则又把这个函数对象存入了实例对象的`__dict__`属性字典中，最终形成了** 循环引用**，因此会造成内存泄漏。
> 题外话：
> 从上面可知，每次调用 a.FuncA 都会生成一个临时的方法对象，因此下面的使用方式会带来一些额外的性能开销：
> for i in range(100000):
> a.FuncA()
> 要实现更高的性能可修改为：
> f = a.FuncA
> for i in range(100000):
> f()

## a.obj的查找顺序
再来看问题 2，obj 是一个实现了`__get__`和`__set__`魔法函数的对象，即符合数据描述符的定义。后面的`a.obj`调用的是`obj.__get__`魔法函数，而前面的则调用的是`obj.__set__`魔法函数，并不会把 obj 对象保存到实例对象的属性字典中，因此不会有内存泄漏问题。
```python
>>> a.obj = a.obj
descr.__get__
descr.__set__
>>> print("obj" in a.__dict__)
False
```
如果把`__set__`注释掉呢，实际就变成了和`a.FuncA`一样的问题，就会有内存泄漏问题
