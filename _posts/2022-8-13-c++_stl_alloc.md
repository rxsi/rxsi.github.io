---
layout: post
title: STL空间配置器
date: 2022-8-13 15:49:03 +0800
categories: C++
tags: C++ 内存池 
author: Rxsi
---

* content
{:toc}

## STL 的内存管理方式
在 C++ 中，使用 new/delete 方式进行对象内存管理分配和回收，底层分为两个步骤：

- new 操作步骤：
   - 1）调用::operator new 配置内存；
   - 2）调用对象的构造函数构造对象

- delete 操作步骤：
   - 1）调用对象析构函数；
   - 2）调用::operator delete 释放内存
<!--more-->

而在 STL 中，将这个过程拆分为了两个独立的语句控制，即内存分配由空间配置器 alloc::allocate() / alloc::deallocate() 负责，对象构造由::construct() / ::destroy() 负责。而空间配置器根据异常处理方式、是否内置轻量内存池（应对内存碎片问题）、多线程下的内存分配处理（内存池的互斥访问）等的不同，分为了第一级配置器和第二级配置器
### 第一级配置器：__malloc_alloc_template
只是对系统 malloc, realloc, free 函数的简单封装，并考虑了分配失败后的异常处理情况
```cpp
//异常处理/*tihs program is in the file of stl_alloc.h*///line 109 to 118
class __malloc_alloc_template {

private://内存不足异常处理
  static void* _S_oom_malloc(size_t);
  static void* _S_oom_realloc(void*, size_t);

#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
  static void (* __malloc_alloc_oom_handler)();
#endif
  //line 141 to 146
  //指定自己的异常处理
  static void (* __set_malloc_handler(void (*__f)()))()
  {
    void (* __old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = __f;
    return(__old);
  }//line 152 to 155
#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
template <int __inst>
void (* __malloc_alloc_template<__inst>::__malloc_alloc_oom_handler)() = 0;
#endif
//line 41 to 50
#ifndef __THROW_BAD_ALLOC
#  if defined(__STL_NO_BAD_ALLOC) || !defined(__STL_USE_EXCEPTIONS)
#    include <stdio.h>
#    include <stdlib.h>//默认的强制退出
#    define __THROW_BAD_ALLOC fprintf(stderr, "out of memory\n"); exit(1)
#  else /* Standard conforming out-of-memory handling */
#    include <new>//抛出用户设计异常处理例程
#    define __THROW_BAD_ALLOC throw std::bad_alloc()
#  endif
#endif
```

### 第二级配置器：__default_alloc_template
这是 STL 容器的默认配置器。
相比较于第一级配置器，增加了轻量级的内存池和多线程环境下内存池的互斥访问（加锁，意味着线程安全）。**当申请的内存大于 128bytes 时移交给第一级配置器处理，而小内存则利用内存池管理分配和回收。主要由自由链表和内存池构成，自由链表负责链接已经申请过的或者重新回收的大小为8的整数倍的空闲内存块，而内存池负责存储当链表没有对应空闲大小的节点时，向堆重新申请更多的2倍以上的内存，然后尽量分配给对应链表20个节点大小的内存。**
当判定要从`free_list`中获取时，会将小额区块的大小调整为`8`的整数倍，比如 30 则调整为 32。
`free_list`的数据结构为：
```cpp
union obj 
{
     union obj * free_list_link;     // 下一个节点的指针
     char client_data[1];            // 内存首地址，柔性数组
}
```
系统同时维护了`16`个`free_list`，分别负责管理8、16、24、32、40、48、56、64、72、80、88、96、104、112、120、128 bytes 大小的区块

## allocate() 流程

1. 判断用户申请的空间大于 128bytes，调用第一级配置器分配空间
2. 若小于等于 128bytes，则检查 free_list，如果有可用区块（每个区块 8bytes），则直接使用，否则调用 refill() 申请新空间

```cpp
/* 内置宏定义
  enum {_ALIGN = 8};
  enum {_MAX_BYTES = 128};
  enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN 
*/

/* __n must be > 0      */
  static void* allocate(size_t __n) // __n是目标内存大小
  {
    void* __ret = 0;

    if (__n > (size_t) _MAX_BYTES) {
      __ret = malloc_alloc::allocate(__n);// 内存大于128时，采用第一级配置器处理
    }
    else {
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n); // 根据传入内存大小，找到链表上的位置，比如__n是32，则_S_freelist_index(__n)返回3，那么_S_free_list+3就可以定位到32所在的链表头地址
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance; // 上锁
#     endif
      _Obj* __RESTRICT __result = *__my_free_list; // 拿到链表头的区块
      if (__result == 0)// 若自由链表free_list不存在可用的区块，则从内存池中填充自由链表
        __ret = _S_refill(_S_round_up(__n));
      else {//若自由链表free_list存在可用区块，调整free_list
        *__my_free_list = __result -> _M_free_list_link; // 链表头指向下一个区块
        __ret = __result;
      }
    }

    return __ret;
  };
```
### _S_refill函数
```cpp
/* Returns an object of size __n, and optionally adds to size __n free list.*/
/* We assume that __n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20; // 一次性最多申请20个区块
    char* __chunk = _S_chunk_alloc(__n, __nobjs); // 分配起始位置的地址
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;

    if (1 == __nobjs) return(__chunk); // 如果数量为1，直接返回当前块
    __my_free_list = _S_free_list + _S_freelist_index(__n); // 指向数组块的位置，这里先以 __n=8 为例

    /* Build free list in chunk */
      __result = (_Obj*)__chunk; // result加一次可以跳到下个节点
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n); // 使数组中的第一个元素指向第二个chunk
      for (__i = 1; ; __i++) {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);//指向下一个节点
        if (__nobjs - 1 == __i) {//到链表最后节点
            __current_obj -> _M_free_list_link = 0;//使next节点等于nullptr表示最后一个节点
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      }
    return(__result);//返回链表首节点地址
}

```
### _S_chunk_alloc函数
```cpp
/* We allocate memory in large chunks in order to avoid fragmenting     */
/* the malloc heap too much.                                            */
/* We assume that size is properly aligned.                             */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{//还是以nobjs为20，size为8来假设
    char* __result;
    size_t __total_bytes = __size * __nobjs;//160
    size_t __bytes_left = _S_end_free - _S_start_free;// 两者初始化都为0 

    if (__bytes_left >= __total_bytes) { // 此时容量已经足够
        __result = _S_start_free;
        _S_start_free += __total_bytes; // 把总容量分出来__total_bytes，然后剩余的仍留在内存池
        return(__result);
    } else if (__bytes_left >= __size) { // 容量虽然不够，但是满足一个size大小，那么计算当前容量满足多少个__nobjs，然后返回。这里的处理是安全的，因为在上层调用时，我们一次性申请的是20个块，而实际是只需要一个，所以这里尽可能的返回就ok了。
        __nobjs = (int)(__bytes_left/__size); 
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else { // 当内存池的容量连一个size的大小都不满足
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4); // 一次性申请2倍的大小，即在假设中计算出来的__bytes_to_get=320 + _S_heap_size >> 4，_S_heap_size值初始为0
        // Try to make use of the left-over piece.
        if (__bytes_left > 0) { // 原内存池还有余量，把这个余量放到链表中，因为在这一次的申请中已经不会被用到了，那么就放到链表中，不要占用当前内存池标志(_S_start_free和_S_end_free)
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        _S_start_free = (char*)malloc(__bytes_to_get); // 调用malloc申请目标大小的内存，此处是320个字节
        if (0 == _S_start_free) {//如果开辟内存失败
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
            _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (__i = __size;
                 __i <= (size_t) _MAX_BYTES;
                 __i += (size_t) _ALIGN) {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if (0 != __p) { // 尝试在链表上找从比当前块更大的没有使用的块，比如当前size是8，但是对应的链表是空的，而且malloc申请失败了，那么就逐个找16、24、32...直到找到一个可用的
                    *__my_free_list = __p -> _M_free_list_link; // 把对应的块从原链表中拿出来，然后增长_S_start_free和_S_end_free，这意味着把这个块放回到了内存池
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs)); // 再尝试匹配，注意此时内存池的空间容量不一定就大于__size*__nobjs，所以是一个递归调用
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
            _S_end_free = 0; // malloc和从链表找合适的未使用的块都失败了，再调用第一级的配置器，这里预设一定会成功。。。
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        // 申请2倍空间成功了。
        _S_heap_size += __bytes_to_get;//此时 _S_heap_size=320
        _S_end_free = _S_start_free + __bytes_to_get;//_S_end_free是 char*类型，开辟一块320字节的内存块
        return(_S_chunk_alloc(__size, __nobjs)); // 递归调用
    }
}

```
## deallocate() 流程

1. 判断回收的内存空间大于 128bytes，调用第一级配置器，最终会使用 free() 函数释放该内存空间
2. 小于等于 128bytes，将其放回到 free_list
```cpp
  /* __p may not be 0 */
  static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)//内存大于128时，采用第一级配置器处理
      malloc_alloc::deallocate(__p, __n);
    else {// 否则，找到相应的自由链表位置，将其回收
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n); // 找到链表中的位置
      _Obj* __q = (_Obj*)__p; // __p是内存块

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance; // 加锁
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list; // 把内存块作为链表的首个块链接入链表中
      *__my_free_list = __q;
      // lock is released here
    }
  }
```
## 内存申请与释放流程

1. 使用`allocate`向内存池申请`size`大小的内存空间，如果请求的大小大于`128 bytes`，则直接使用第一级空间配置器，背后的逻辑就是`malloc`、`free`、`realloc`
2. 如果请求内存小于`128 bytes`，则先将`size`调整为`8`的倍数，然后根据`size`值查找到合适的自由链表
   1. 如果链表不为空，则返回第一个节点，修改第二个节点为头节点
   2. **如果链表为空，则使用**`**refill**`**请求分配一个新的节点**
      1. 如果内存池中有多于一个节点的空间，则分配尽可能多的节点（最多 20 个），并将第一个节点返回，其余节点加入到链表中
      2. 如果内存池连一个节点的空间都没有了，则向操作系统请求分配新内存
         1. 如果分配成功，再次执行 **b** 过程
         2. 如果分配失败，则查询比所要申请大小更大的各个自由链表，尝试利用他们的剩余空间
            1. 如果找到空间，再次执行 **b** 过程
            2. 找不到，则抛出异常
3. 当使用`deallocate`释放内存空间时，如果释放的空间大于`128 bytes`，则直接调用`free`方法
4. 否则按照大小放回到对应的自由链表中

## construct() 和 destroy()
construct() 负责对象的构造，而 destroy() 负责对象的析构，二者都不进行内存空间的操作
## uninitialized_copy(),uninitialized_fill()和uninitialized_fill_n()

- uninitialized_copy() 负责向未初始化的内存空间拷贝填充某个已有数据
- uninitialized_fill() 负责向未初始化的内存空间填充某个初值
- uninitialized_fill_n() 负责向未初始化的内存空间填充n个初值

```cpp
#ifndef __SGI_STL_INTERNAL_UNINITIALIZED_H
#define __SGI_STL_INTERNAL_UNINITIALIZED_H

__STL_BEGIN_NAMESPACE

// uninitialized_copy

// Valid if copy construction is equivalent to assignment, and if the//  destructor is trivial.
template <class _InputIter, class _ForwardIter>/*该函数接受三个迭代器参数：迭代器first是输入的起始地址，
*迭代器last是输入的结束地址，迭代器result是输出的起始地址
*即把数据复制到[result,result+(last-first)]这个范围
*为了提高效率，首先用__VALUE_TYPE()萃取出迭代器result的型别value_type
*再利用__type_traits判断该型别是否为POD型别
*/
inline _ForwardIter
  uninitialized_copy(_InputIter __first, _InputIter __last,
                     _ForwardIter __result){
  return __uninitialized_copy(__first, __last, __result,
                              __VALUE_TYPE(__result));}

template <class _InputIter, class _ForwardIter, class _Tp>
inline _ForwardIter
/*利用__type_traits判断该型别是否为POD型别*/__uninitialized_copy(_InputIter __first, _InputIter __last,
                     _ForwardIter __result, _Tp*){
  typedef typename __type_traits<_Tp>::is_POD_type _Is_POD;
  return __uninitialized_copy_aux(__first, __last, __result, _Is_POD());}

template <class _InputIter, class _ForwardIter>
_ForwardIter //若不是POD型别，就派送到这里__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __false_type){
  _ForwardIter __cur = __result;
  __STL_TRY {//这里加入了异常处理机制
    for ( ; __first != __last; ++__first, ++__cur)
      _Construct(&*__cur, *__first);//构造对象，必须是一个一个元素的构造，不能批量
    return __cur;
  }
  __STL_UNWIND(_Destroy(__result, __cur));//析构对象}

template <class _InputIter, class _ForwardIter>
inline _ForwardIter //若是POD型别，就派送到这里__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __true_type){
        /*调用STL的算法copy()
        *函数原型：template< class InputIt, class OutputIt >
        * OutputIt copy( InputIt first, InputIt last, OutputIt d_first );
        */
        return copy(__first, __last, __result);}//下面是针对char*，wchar_t* 的uninitialized_copy()特化版本
inline char* uninitialized_copy(const char* __first, const char* __last,
                                char* __result) {/* void* memmove( void* dest, const void* src, std::size_t count );
* dest指向输出的起始地址
* src指向输入的其实地址
* count要复制的字节数
*/
  memmove(__result, __first, __last - __first);
  return __result + (__last - __first);}

inline wchar_t* 
uninitialized_copy(const wchar_t* __first, const wchar_t* __last,
                   wchar_t* __result){
  memmove(__result, __first, sizeof(wchar_t) * (__last - __first));
  return __result + (__last - __first);}

// Valid if copy construction is equivalent to assignment, and if the// destructor is trivial.
template <class _ForwardIter, class _Tp>/*若是POD型别，则调用此函数
        */
inline void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __true_type){/*函数原型：template< class ForwardIt, class T >
  * void fill( ForwardIt first, ForwardIt last, const T& value );  
  */
        fill(__first, __last, __x);}

template <class _ForwardIter, class _Tp>/*若不是POD型别，则调用此函数
        */
void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __false_type){
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __cur != __last; ++__cur)
      _Construct(&*__cur, __x);
  }
  __STL_UNWIND(_Destroy(__first, __cur));}

template <class _ForwardIter, class _Tp, class _Tp1>//用__type_traits技术判断该型别是否
为POD型别
inline void __uninitialized_fill(_ForwardIter __first, 
                                 _ForwardIter __last, const _Tp& __x, _Tp1*){
  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;
  __uninitialized_fill_aux(__first, __last, __x, _Is_POD());
                   
}

template <class _ForwardIter, class _Tp>/*该函数接受三个参数：
*迭代器first指向欲初始化的空间起始地址
*迭代器last指向欲初始化的空间结束地址
*x表示初值
*首先利用__VALUE_TYPE()萃取出迭代器first的型别value_type
*然后用__type_traits技术判断该型别是否为POD型别
*/
inline void uninitialized_fill(_ForwardIter __first,
                               _ForwardIter __last, 
                               const _Tp& __x){
  __uninitialized_fill(__first, __last, __x, __VALUE_TYPE(__first));}

// Valid if copy construction is equivalent to assignment, and if the//  destructor is trivial.
template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter
        /*若是POD型别，则调用此函数
        */__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __true_type){
  /*调用STL算法
  *原型：template< class OutputIt, class Size, class T >
  * void fill_n( OutputIt first, Size count, const T& value );
  * template< class OutputIt, class Size, class T >
  * OutputIt fill_n( OutputIt first, Size count, const T& value );
  */
        return fill_n(__first, __n, __x);}

template <class _ForwardIter, class _Size, class _Tp>
_ForwardIter
        /*若不是POD型别，则调用此函数
        */__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __false_type){
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __n > 0; --__n, ++__cur)
      _Construct(&*__cur, __x);
    return __cur;
  }
  __STL_UNWIND(_Destroy(__first, __cur));}

template <class _ForwardIter, class _Size, class _Tp, class _Tp1>
inline _ForwardIter 
        //用__type_traits技术判断该型别是否为POD型别__uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x, _Tp1*){
  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;
  //_Is_POD()判断value_type是否为POD型别
  return __uninitialized_fill_n_aux(__first, __n, __x, _Is_POD());}

template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter 
/*该函数接受三个参数：
*迭代器first指向欲初始化的空间起始地址
*n表示欲初始化空间大小
*x表示初值
*首先利用__VALUE_TYPE()萃取出迭代器first的型别value_type
*然后用__type_traits技术判断该型别是否为POD型别
*/uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x){
  //__VALUE_TYPE(__first)萃取出first的型别value_type
        return __uninitialized_fill_n(__first, __n, __x, __VALUE_TYPE(__first));}

__STL_END_NAMESPACE

#endif /* __SGI_STL_INTERNAL_UNINITIALIZED_H */

// Local Variables:// mode:C++// End:
```
