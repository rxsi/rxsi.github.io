---
layout: post
title: Python热更原理
date: 2022-03-25 13:43:55 +0800
categories: Python
tags: python 热更新 
author: Rxsi
excerpt: 
---

* content
{:toc}

### 模块导入流程

- 模块导入流程： 
   1. 当导入一个模块时，首先从 **sys.modules** 中查找，如果查找到目标模块对象，则返回模块对象，查找结束。
   2. 如果未能查找到，则从 **sys.path** 中定义的路径开始查找，当查找到模块文件之后，将模块文件实例化为模块对象，并存入 **sys.modules** 中，返回模块对象，查找结束。
   3. 如果依然没有查找到，则抛异常，查找结束

> 结论：导入模块会生成模块对象，且全局唯一，存放于 sys.modules 中


### 模块导入方式的差异

```python
# <A.py>
class Base():
	def __init__(self, desc):
		print(f"{desc} Init!")
		self.num = 0

	def AddNum(self):
		self.num += 1

	def PrintVer(self):
		print("Base Ver 1.0")

	def PrintNum(self):
		print(f"Base Num: {self.num}")

if "g_SingleClass" not in globals():
	g_SingleClass = Base("g_SingleClass")

no_SingleClass = Base("no_SingleClass")

def Method():
	print("Method Version 1.0")
```

以下是几种导入方式：

```python
import A
import A as ModuleA
from A import Base
from A import Base as ClassBase
from A import *
```

1. 当直接导入 A 模块名的时候，会在 B 模块的全局命名空间加入 A 模块对象的引用，因此可通过 A.xxx 访问 A 模块对象中的属性
2. 通过 from A import xxx 时，会在 B 模块的全局命名空间加入 A 模块对应属性对象的引用，可直接通过 xxx 使用导入的属性，并不导入 A 模块对象的引用，即 A.xxx 是非法的访问
3. from A import * ，会将 A 中所有的属性都导入，本质上与第2点相同
4. as 只是在存入引用的时候用了别名，本身遵循前两点的导入规则

> 结论：不同的导入方式，会导致不同的引用绑定方式


### reload函数

从模块的导入过程可以知道，当我们对修改后的模块进行 import 时，是不能使新修改生效的，需要先删除 sys.modules 内的旧模块对象，才能执行新的 import。
演示：

```python
# 在 Python 解释器中执行：

import A
A.Method()
# 输出：Method Version 1.0

# 修改Version 为 2.0
import A
A.Method()
# 输出：Method Version 1.0

del sys.modules["A"]
import A
A.Method()
# 删除旧模块对象后，重新导入可以使新属性生效 Method Version 2.0
```

使用这种方式有以下几个缺点：

1. 新旧模块具有不同的地址，这样我们需要把导入该模块的所有地方都进行修改，比较麻烦
2. 会删除旧的命名空间，我们有时候会在模块中定义全局唯一的对象，比如定义 g_SingleClass = Base()，当有额外对这个唯一对象的引用时，销毁当前的命名空间就不会销毁这个唯一对象。然后又重新生成新的对象，所以会造成同时共存两个 g_SingleClass，造成异常

Python 提供了一个内置的模块重载函数：reload，有以下特点：

1. 保持原模块对象的地址不变，即不改变 sys.modules 中对应模块的地址
2. 原模块下的所有类、方法、属性等都会被重新生成，即原模块**dict** 中的属性会拥有新的对象地址
3. 不删除原命名空间，而是产生新的同名对象覆盖。由于这个特性，如果对原模块的改动是删除某个属性，那么 reload 之后还是可以访问到该旧属性。如果是新增属性，则 reload 之后可以正常访问到新属性

演示：

```python
# 在 Python 解释器中执行：

import A
A.Method()
# 输出：Method Version 1.0
id(A)
# 模块地址，输出： 2062835254272

# 修改Method Version 为 2.0，进行重新 reload
from importlib import reload
reload(A)

import A
A.Method()
# 输出：Method Version 2.0
id(A)
# reload 之后模块的地址保持不变，输出： 2062835254272
```

从上面的演示，可以知道对于直接 import 模块而言，reload 之后可以使通过该模块引用访问的函数生效，可以达到热更的效果。
但是对于 from A import xxx 而言，因为直接保存的是属性的地址，因此并不能访问到修改后的新属性

演示：

```python
# 在 Python 解释器中执行：

from A import Method
import A
Method()
# 直接访问Method，输出：Method Version 1.0
A.Method()
# 通过A.Method访问，输出：Method Version 1.0

id(Method) == id(A.Method)
# 地址相同，属于同一个函数对象，输出 True

# 修改Method Version 为 2.0，进行重新 reload
from importlib import reload
reload(A)

Method()
# 函数输出没有改变，输出：Method Version 1.0
A.Method()
# 函数输出改变了，输出：Method Version 2.0

id(Method) == id(A.Method)
# 地址发生改变，输出 False
```

由于 reload 会重新生成新的属性对象，对旧的命名空间进行覆盖，因此当我们不希望重新生成对象时，可以使用 "xxx" not in globals() 进行判断

演示：

```python
# 在 Python 解释器中执行：

# 当第一次import 时，
import A
# 输出：
# g_SingleClass Init!
# no_SingleClass Init!

reload(A)
# 输出：
# no_SingleClass Init!
```

> 结论：
> 1. 使用 reload 可以保持模块对象地址不变，因此不需要处理模块对象引用
> 2. 当导入的方式是 form ... import xxx 时，需要进行额外处理
> 3. 对于不希望reload之后重新生成的对象，应该使用 **"xxx" not in globals()** 进行拦截

### 热更处理

根据对象类型的不同，需要处理的类型可以分为：

1. 类，使用 isinstance(a, type) 判断
2. 函数，使用 isinstance(a, types.FunctionType) 判断
3. 属性描述符，使用 isinstance(a, property)
4. 动态绑定的方法, 使用 isinstance(a, types.MethodType)
5. 静态函数，isinstance(a, staticmethod)
6. 类函数，isinstance(a, classmethod)

> 注：还有些由 C 转换而来的函数类型，使用 types.MemberDescriptorType, types.GetSetDescriptorType)，这个源码中直接调用的是 C_reload 方法，不清楚具体实现，不讲解


已知 reload 之后会生成新的对象，现在有两种处理方式：

1. 使用新生成的对象，去替换旧的对象，那么需要查找到哪些模块引用了这些旧对象，然后逐个替换。但是我目前没有查到有相关方法支持查询对象被谁引用，因此这个方案执行难度大。
2. 用新生成的对象的属性去替换替换旧对象的属性，然后用旧对象替换新生成的对象。这样就可以不用考虑哪些模块导入保存了旧对象的引用，又可以成功替换成新属性，处理起来会更加高效。

#### 保留旧命名空间的属性对象

```python
if old_objects is None:
    old_objects = {}

# collect old objects in the module
for name, obj in list(module.__dict__.items()):
    if not append_obj(module, old_objects, name, obj):
        continue
    key = (module.__name__, name)
    try:
        old_objects.setdefault(key, []).append(weakref.ref(obj)) # 弱引用保存对象
    except TypeError:
        pass
```

#### 执行reload，重生成各属性对象

```python
try:
    module = reload(module)
except:
    # restore module dictionary on failed reload
    module.__dict__.update(old_dict) # old_dict是reload之前对原module的__dict__属性的拷贝，当reload失败时，进行回滚
    raise
```

#### 各对象处理方法

```python
UPDATE_RULES = [
    (lambda a, b: isinstance2(a, b, type), update_class),
    (lambda a, b: isinstance2(a, b, types.FunctionType), update_function),
    (lambda a, b: isinstance2(a, b, property), update_property),
]
UPDATE_RULES.extend(
    [
        (
            lambda a, b: isinstance2(a, b, types.MethodType),
            lambda a, b: update_function(a.__func__, b.__func__),
        ),
    ]
)


# 当a、b新旧对象都通过type_check后，调用update方法进行更新
def update_generic(a, b):
    for type_check, update in UPDATE_RULES:
        if type_check(a, b):
            update(a, b)
            return True
    return False
```

#### 类处理

```python
def update_class(old, new):
    # 遍历更新旧类对象中的所有属性对象
    for key in list(old.__dict__.keys()):
        old_obj = getattr(old, key)
        try:
            new_obj = getattr(new, key)
            if (old_obj == new_obj) is True:
                continue
        except AttributeError:
            try:
                # 步骤1：删除已经不存在于新类对象中的属性
                delattr(old, key)
            except (AttributeError, TypeError):
                pass
            continue
        # 步骤2：检测新旧属性对象是否还可以进一步更新，比如都是函数、静态函数等
        if update_generic(old_obj, new_obj):
            continue

        try:
            # 步骤3：对于已不可进一步更新的旧属性，直接替换为新属性
            setattr(old, key, getattr(new, key))
        except (AttributeError, TypeError):
            pass  # skip non-writable attributes

    for key in list(new.__dict__.keys()):
        if key not in list(old.__dict__.keys()):
            try:
                # 步骤4：将只存在于新类对象中的属性，填充到旧类对象中
                setattr(old, key, getattr(new, key))
            except (AttributeError, TypeError):
                pass  # skip non-writable attributes
```

#### 函数处理

```python
def update_function(old, new):
    # 步骤1：检查闭包内容是否修改：
    # 判断条件：old.__closure__ != new.__closure__
    # 闭包参数是存储在 PyCellTypeObject 对象中，在Python层面无法直接修改，因此使用引擎层 C_reload 进行修改
    
    for name in func_attrs:
        try:
            # 步骤2：将旧函数对象的属性替换为新属性
            setattr(old, name, getattr(new, name))
        except (AttributeError, TypeError):
            pass
```

#### 属性描述符处理

```python
def update_property(old, new):
    """Replace get/set/del functions of a property"""
    # 分别对 fdel, fget, fset 三个属性进行更新
    update_generic(old.fdel, new.fdel)
    update_generic(old.fget, new.fget)
    update_generic(old.fset, new.fset)
```

#### 动态绑定方法处理

- 什么是动态绑定方法？

```python
def desc(self):
    print(self)

base = A.Base("test");
base.dec = types.MethodType(desc, base)
base.desc()
# 输出：<A.Base object at 0x000001FF806E6B80>
```

```python
# function对象隐藏在__func__属性中
def update_method(old, new):
    update_function(old.__func__, new.__func__)
```

#### 静态方法 和 类方法 处理

```python
# function对象隐藏在__func__属性中
def update_method(old, new):
    update_function(old.__func__, new.__func__)
```

#### 对象替换

经过上面替换步骤，原本旧的对象都有了新的属性
最后的步骤就是将module对象里面的新生成的对象替换成原先旧的对象

```python
setattr(module, name, old)
```

#### 热更完成之后

可以执行必要的操作

```python
if hasattr(mod, "Init" and mod.Init.__module__ == mod.__name__:
    mod.Init
```

### 不能热更的情况

#### 类热更前后的 __slots__ 不一致

实例代码：

```python
class Base():
    __slots__ = "num", "name"
    def __init__(self):
        self.num = 0
        self.name = ""
```

```python
# 在 Python 解释器中执行：

# 当没有定义 __slots__ 时
import A
A.Base.__dict__
# 输出：
mappingproxy({'__module__': 'A',
              '__init__': <function A.Base.__init__(self)>,
              '__dict__': <attribute '__dict__' of 'Base' objects>,
              '__weakref__': <attribute '__weakref__' of 'Base' objects>,
              '__doc__': None})
              
 # 添加 __slots__ 然后重新reload之后打印
 A.Base.__dict__
 mappingproxy({'__module__': 'A',
              '__slots__': ('num', 'name'),
              '__init__': <function A.Base.__init__(self)>,
              'name': <member 'name' of 'Base' objects>,
              'num': <member 'num' of 'Base' objects>,
              '__doc__': None})
```

从上面的例子可以看到，当定义**slots**之后，属性会被生成为 **Member对象** 直接存储在类对象上，**内存是连续的**，这会占用实际的类对象的内存空间。
当访问这些属性时，是通过偏移值信息去直接计算该 Member对象 在类内存上的位置。
也就是说当一个旧的类对象创建之后，他为这些**slots**分配的内存空间就已经是固定好的了，因此并不能做到在不修改原类对象地址的情况下，进行**slots**修改后的扩容/缩容。

#### metaclass 元类

实例代码：

```python
class Meta(type):
    pass

class Base(metaclass=Meta):
    __slots__ = "num", "name"
    def __init__(self):
        self.num = 0
        self.name = ""
```

```python
# 在 Python 解释器中执行：

import A
type(A.Base)
# 输出：A.Meta
```

参考 Github Pydev项目的处理方式，当判断到是元类类型时，直接把他当成 **正常类对象** 处理即可

```python
# New: dealing with metaclasses.
if hasattr(newobj, '__metaclass__') and hasattr(newobj, '__class__') and newobj.__metaclass__ == newobj.__class__:
      self._update_class(oldobj, newobj)
      return
```

#### __init__ 方法定义了新的属性，由旧类实例化的对象没有创建新属性

实例代码：

```python
# A.py
class Base():
    def __init__(self):
        self.num = 0
        self.name = ""

# B.py
from A import Base

a = Base()
def show():
    print(a.__dict__)


# 热更前调用 B.show()
# 输出： {'num' = 0, 'name' = ""}

# 给 A.Base.__init__ 添加 self.age = 0 后热更再调用 B.show()
# 输出：{'num' = 0, 'name' = ""}
```

处理方法有：

1. 当对象获取属性时，使用 getattr()/ setattr() 方法，当获取不到新添加的属性时，手动添加属性。这种方法治标不治本。
2. 更新前使用 **gc.get_referrers(old_class)** 找到之前创建的所有对象，然后创建新属性的初始值
示例代码： 
```python
def update_class(old_class, new_class):
  ...
  if hasattr(old_class, '_on_reload_instance'):
    for x in gc.get_referrers(old_class):
      if isinstance(x, old_class):
        x._on_reload_instance()
```
