---
layout: post
title: C++原子性和内存模型
date: 2022-8-16 15:49:03 +0800
categories: C++
tags: C++ atomic volatile 
author: Rxsi
---

* content
{:toc}

# CPU架构
现在处理器架构一般为下图的结构：

![cpu_structure.png](/images/c%2B%2B_atomic_volatile/cpu_structure.png)
<!--more-->
## 缓存一致性
在多核结构下，每个核心有多级独占的 cache 高速缓冲，通常为了保证缓冲间数据的一致性，因此有了 **MESI协议**。该协议通过把 cacheline 标记为4个状态之一，形成状态机，从而保证缓存一致性。
状态转换如下图：

![MESI.png](/images/c%2B%2B_atomic_volatile/MESI.png)

注：如果发生不同核心同时对某个变量进行相同操作，则总线会参与仲裁协调，以保证只有其中一个核的操作有效
## 内存一致性（内存模型）
CPU 与内存之间的数据传输关系可以归纳为：

- Load：将数据从内存中加载到寄存器
- Store：将寄存器中的数据回写到内存中

内存一致性描述的就是 store、load 之间的执行顺序，根据执行顺序的区别有 4 种模型：

- Sequential Consistency（顺序一致性）：多核之间的store、load完全不会乱序执行，最严格的的模型
- Total Store Order（完全存储定序）：store-load 之间在不影响单线程逻辑的前提下可以乱序，但是 store-store 之间保持顺序
- Part Store Order（部分存储定序）：在 total store order 的基础上，不保证 store-store 之间的顺序
- Relax Memory Order（宽松存储定序）：store 和 load的顺序完全不保证

# 存储缓存（Store Buffer）
多核之间的数据同步是通过 **MESI** 协议同步的，但是缓存的一致性消息传递是要时间的，这就使其切换时会产生延迟。当一个缓存被切换状态时其他缓存收到消息完成各自的切换并且发出回应消息这么一长串的时间中 CPU 都会等待所有缓存响应完成。可能出现的阻塞都会导致各种各样的性能问题和稳定性问题。
比方说 CPU0 准备对 变量a 进行写操作，根据 MESI协议 它会向其他所有的核心发送消息，使其他核心的 cacheline 如果有保存变量 a 则标记为 **Invalid**，并返回 Ack 信息，此时 CPU0 才会对变量进行 Modify 操作。这个过程，CPU0 就会阻塞等待到消息返回才会进行下一步的处理。

![no_store_buffer.png](/images/c%2B%2B_atomic_volatile/no_store_buffer.png)

当 CPU0 准备对 变量a 写操作时，其实其他核心保存的 变量a 信息已经无关紧要了，毕竟都会被重新覆盖掉，而目前的设计中会使 CPU0 一直阻塞直到收到 Ack 回复。 为了避免这种 CPU 运算能力的浪费，Store Bufferes 被引入使用。处理器把它想要写入到主存的值写到缓存，然后继续去处理其他事情，直到收到 Ack 信息，再把数据写回到内存。

![store_buffer.png](/images/c%2B%2B_atomic_volatile/store_buffer.png)

## Store Forwarding
在引入 store buffer 前，load、store 操作都是 CPU 直接与 cache 交互，因此单核 CPU 上程序的执行流程符合 program order。但是 引入 store buffer 之后，由于 store 操作可能缓存在 buffer 中，因此打乱了单核 store-load 的执行顺序。
为了解决这个问题，引入了 store forwarding，当 CPU 执行 load 操作时，不但要看 cache，还要看 store buffer 中的内容，当buffer 中有数据时，则采用该数据。

![store_forwarding.png](/images/c%2B%2B_atomic_volatile/store_forwarding.png)

## Invalidate Queues
引入 Store buffer 之后，解决了发送端 CPU 等待 ACK 信息而阻塞的问题，但是当接收端 CPU 无法及时处理发送端信息（Invalidate Message，使本 CPU 的 cacheline 数据状态转变为 Invalid）时，最终会导致发送端 CPU 的 store buffer 因不能及时接收到 ACK 信息而最终被塞满。
为了解决这个问题，引入了 Invalidate Queues，用以缓存 Invalidate Message，并直接回复 ACK 信息，表示自己已经接收到，并再慢慢处理。
## Store Buffer 和 Invalidate Queues 的风险
虽然加入了 Store Buffer，但是因为有 store forwarding，因此可以保证在单核情况下的程序执行流程，但是对于多核来说，这个流程就无法保证了，这个问题是无法通过 MESI 协议解决的。再加上 Invalidate Queues 会使得 CPU 处理 Invalidate Message 的时机出现延后，如果不加措施保障 CPU 在处理 Message 之前不能操作相关数据的 cacheline，那么也会造成读取的数据不是最新的。
总的来说就是各 CPU 核心在处理 Store、Load 指令时，可能存在重排的现象（乱序执行，对于本核来说乱序执行之后的结果仍是正确的，但是对于其他核依赖于这份数据的则会造成异常），归纳为 **Load-Store、Store-Load、Load-Load、Store-Store** 这四种乱序结果。
不同架构的处理器有不同的内存模型（CPU MEMORY MODEL），在不同的数据模型下对于 **Load/Store** 操作的顺序有着不同的处理方式，概括如下：

![memory_order.png](/images/c%2B%2B_atomic_volatile/memory_order.png)

此外，在 **单核CPU** 下，**单次**的 Load 和 Store 在满足 **内存对齐** 的条件下是原子的，不需要额外加锁。如`int a = 1`具有原子性，而`a++`则是多次内存访问，不具有原子性。
**内存对齐的意义：**

1. 提升CPU读取效率
2. 保证单次 Reader 和 Writer的原子性：CPU对单次字长的操作可以保证原子性
3. 增加了内存的消耗：位域的出现可以灵活利用内存空间

在不同位数系统中，CPU 单次最大处理位数不同，因此内存对齐有不同的结果
```cpp
class A
{
    int a;
    char b;
    double c;
    char d;
}
// 在 64 位系统中，最大处理位数是 64 位，即 8 Bytes
// 因此上述的结果为 4 + 1 + (3) + 8 + 1 + (7) = 24
    
// 而在 32 位系统中，最大处理位数是 32 位，即 4 Bytes，因此实际上double类型是分为了两个 4Bytes 存储
// 因此上述结果为 4 + 1 + (3) + 4 + 4 + 1 + (3) = 16
```
在 **多核CPU** 下，单核CPU 的原子性仍然有保障，但是多核之间的原子性则不成立，因此也需要额外的保护措施才能实现多核间的原子性。
现代 **编译器** 还会对原始代码进行优化和重排，目的是在不改变原有语义的情况下，加速程序运行。但是这种优化编译器只能保证单个执行流中的语义不变，无法判断其他执行流对本执行流的依赖关系，因此在这种情况下，优化和重排均可能会带来异常。
因此为了程序的最终正确性，我们分别需要 **优化屏障 和 内存屏障** 两个机制来解决编译器和 CPU 的乱序执行情况。
# linux内核
## 优化屏障
在 linux 内核中，使用 barrier() 和 ACCESS_ONCE() 抑制编译器优化
## 内存屏障

1. 通用 barrier，保证读写操作有序的，mb() 和 smp_mb()
2. 写操作 barrier，仅保证写操作有序的，wmb() 和 smp_wmb()
3. 读操作 barrier，仅保证读操作有序的，rmb() 和 smp_rmb()

# Volatile、atomic
C++ 中的 **volatile** 关键字、**std::atomic** 变量均是为了避免内存访问过程中出现一些不符合预期的行为。

概括如下：

|  | volatile | std::atomic |
| --- | --- | --- |
| 抑制编译器重排 | Yes | Yes |
| 抑制编译器优化 | Yes | Yes |
| 抑制 CPU 乱序 | No | Yes |
| 保证访问原子性 | No | Yes |

> 注：在线实时汇编转换网站：[https://www.godbolt.org/](https://www.godbolt.org/)

## Volatile
### 抑制编译器重排

示例1：两个值都非volatile
```cpp
int a;
int b;
int main()
{
    a = b + 1;
    b = 0;
}
 
 // 转换后的汇编码如下：
mov     eax, DWORD PTR b[rip]
mov     DWORD PTR b[rip], 0 //先计算了b=0,发生了重排
add     eax, 1
mov     DWORD PTR a[rip], eax
```
示例2：其中一个值为volatile类型
```cpp
int a;
volatile int b;
int main()
{
    a = b + 1;
    b = 0;
 }
 
 // 转换后的汇编码如下：
mov     eax, DWORD PTR b[rip] // 依然会发生重排,即证明单个属性变量声明为volatile是无用的
mov     DWORD PTR b[rip], 0
add     eax, 1
mov     DWORD PTR a[rip], eax
```
示例3：两个值都是volatile类型
```cpp
volatile int a;
volatile int b;
int main()
{
    a = b + 1;
    b = 0;
 }
 
 // 转换后的汇编码如下：
mov     eax, DWORD PTR b[rip]
add     eax, 1 // 可见没有发生重排
mov     DWORD PTR a[rip], eax
xor     eax, eax
mov     DWORD PTR b[rip], 0
```
**结论：volatile可以在同为volatile变量之间抑制编译器重排**

### 抑制编译器优化
示例1：非volatile类型
```cpp
int main()
{
    int a = 5;
}

// 转换后的汇编码如下：
xor     eax, eax // 因为a值并没有调用，编译器直接优化掉了
```
示例2：volatile类型
```cpp
int main()
{
    volatile int a = 5;
}

// 转换后的汇编码如下：
mov     DWORD PTR [rsp-4], 5 // 未被优化掉,依然是将立即数$0x5压入栈中(rsp是栈顶指针,栈内存是由高往地变化,因此是-0x4,即4字节)
xor     eax, eax
```
**结论：可通过volatile修饰一些不想要编译器优化的语句**

### 对volatile变量的读取每次都从内存中读取最新值
示例1：非volatile变量
```cpp
#include <iostream>
int fun(int num)
{
    return num + 1;
}
int main()
{
    int a = 1;
    int c;
    std::cin >> c;
    a = fun(c);
    int b = a + 1;  // lea     esi, [rbx+1]，直接从寄存器读取a值然后+1
    std::cout << b << std::endl;
}
```
示例2：volatile变量
```cpp
#include <iostream>
int fun(int num)
{
    return num + 1;
}
int main()
{
    volatile int a = 1;
    int c;
    std::cin >> c;
    a = fun(c);
    int b = a + 1;  // mov     esi, DWORD PTR [rsp+4]
                   // add     esi, 1 先从内存中读取最新值，然后再+1
    std::cout << b << std::endl;
}
```
**结论：volatile修饰的变量每次修改后都会从寄存器中刷回内存，每次读取都会从内存读取最新值**

### 与Java的volatile关键字的不同之处
volatile 关键字可以在汇编层面抑制重排，但是现代化的 CPU 在程序运行时会额外优化语句的执行顺序，在这方面 C++ 的 volatile 关键字无能为力。
Java 的 volatile 关键字则可以实现这方面的抑制力，本质上就是多了一层 **happens_before （acquire - release 语义）**，这在 C++ 中是通过 std::atomic 实现的。
## std::atomic
std::atomic 是禁用 **拷贝构造** 函数的，要注意初始化方式
### 六种内存模型
#### memory_order_releaxed：
不保证变量间的执行顺序，只保证当前变量的 store/load 的原子性
因此适用于当个变量的原子性保证。
应用场景：程序计数器
```cpp
#include <atomic>
#include <thread>
#include <assert.h>
#include <vector>
#include <iostream>
// std::atomic<int> cnt = 0; 这种初始化方式报错, atomic(const atomic&) = delete;即禁止拷贝构造
std::atomic<int> cnt(0); // 这种是直接初始化，调用的是类对应的有参构造函数
// int cnt; // 这种是默认初始化，对于函数之外的内置变量初始化值为0,多线程同时修改时线程不安全
void f()
{
    for (int n = 0; n < 1000; ++n)
    {
        // cnt++; // 这也是保证了原子性，因为atomic有重载operator++ 和 operator--，但是默认的内存序是 memory_order_seq_cst（最严格），在此处只需要保证当前++操作的原子性，因此使用memory_order_relaxed即可
        cnt.fetch_add(1, std::memory_order_relaxed); // memory_order_relaxed只保证当前语句的原子性,不保证其他store\load之间的顺序
    }
}
int main()
{
    std::vector<std::thread> v;
    for (int n = 0; n < 10; ++n)
    {
        v.emplace_back(f);
    }
    for (auto& t : v)
    {
        t.join();
    }
    assert(cnt == 10000);
    return 0;
}
```
#### memory_order_consume
影响 load 行为（将值读取到寄存器），处于当前变量的 load 行为之后的且 与**当前变量相关** 的读取行为不会被优化到在该语句之前执行
```cpp
p = ptr.load(std::memory_order_consume);
p += 1; // 此语句永远只会在load之后再执行
a = 123; // 有可能会优化到load之前
```
#### memory_order_acquire
影响 load 行为，处于当前变量的 load 行为之后的所有与 **内存相关** 的操作都不会被优化到该变量之前执行
```cpp
p = ptr.load(std::memory_order_acquire);
p += 1; // 不会优化到load之前
a = 123; // 不会优化到load之前
```
#### memory_order_release：
影响 store 行为，处于当前 store 行为之前的所有与 **内存相关** 的操作都不会优化到该 store 行为之后
```cpp
std::string* p = new std::string("hello"); // 不会优化到store之后执行
data = 33; // 不会优化到store之后执行
ptr.store(p, std::memory_order_release);
```
#### release + comsume 构成了 Carries dependency 关系
```cpp
#include <thread>
#include <atomic>
#include <assert.h>
#include <string>
std::atomic<std::string*> ptr;
int data;
void producer()
{
    std::string* p = new std::string("Hello");
    data = 22;
    ptr.store(p, std::memory_order_release); // store操作属于把数据写回内存，因此需要把前面的内存操作都先执行完，实际上控制的是 store buffer 的顺序
}
void consumer()
{
    std::string* p2;
    while (!(p2 = ptr.load(std::memory_order_consume))); // load操作属于把数据从内存读到寄存器，实际上控制的是 invalid queue 的顺序
    assert(*p2 == "Hello"); // 永远不会失败
    assert(data == 42);// 可能失败,因为可能被提前到while之前做了判断。因为是release，因此while之后是保证data==42的
}
int main()
{
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
}
```
#### release + acquire 构成了 synchronize-with 关系
```cpp
#include <thread>
#include <atomic>
#include <assert.h>
#include <string>
#include <iostream>
std::atomic<bool> ready{false};
int data{0};
std::atomic<int> var{0};
void sender()
{
    data = 22;
    var.store(100, std::memory_order_relaxed);
    ready.store(true, std::memory_order_release); // 本语句之前的内存操作语句都不会被优化到本store之后，控制的是 store buffer 的顺序
}
void receiver()
{
    while(!ready.load(std::memory_order_acquire)); // 在这之后的所有的内存操作都不能在被优化到本load语句之前，控制的是 invalid queue 的顺序
    assert(data == 22); // 永远都成功
    assert(var == 100); // 永远都成功
}
int main()
{
    std::thread t1(sender);
    std::thread t2(receiver);
    t1.join();
    t2.join();
}
```
#### memory_order_acq_rel
同时影响 store-load 的行为，即在他之前的任何内存操作不能优化到他之后，而原本在他之后的语句不能优化到他之前
一般用在 RMW 操作语境，**RMW 操作保证永远读取的都是最新值**
```cpp
#include <thread>
#include <atomic>
#include <assert.h>
#include <vector>
std::vector<int> data;
std::atomic<int> flag{0};
int num;
void thread_1()
{
    data.push_back(42);
    flag.store(1, std::memory_order_release);
}
void thread_2()
{
    // 对于RMW（Read-Modify-Writes，需要实现原子性）操作，属于把三个操作绑定为一个原子性操作，因此需要使用memory_order_acq_rel  ry_order_seq_cst
    // compare_exchange_weak(T& expected, T desired, std::memory_order order = std::memory_order_seq_cst)
    // compare_exchange_strong(T& expected, T desired, std::memory_order order = std::memory_order_seq_cst)
    // 作用都是比较*this和 expected值，如果二者相等，那么把*this替换为 desired，并返回true（RMW 操作）
    // 如果不想等则把expected赋值为*this的值（load 操作）
    // 不同的是weak可能会因为（fail spuriously) 而导致返回错误结果，因此需要放在loop中判断（如while）
    int expected = 1; // 由于memory_order_acq_rel，因此expected会比下面的语句优先执行
    num = 22; // 这个语句也不会被重排到后面执行
    while(!flag.compare_exchange_strong(expected, 2, std::memory_order_acq_rel)) // 是当flag的值与expected不一致时，代表thread_1还没有执行，flag    会使expected被置为0，因此需要在循环体内继续把expected设置为1
    {
        expected = 1;
    }
    assert(num == 22);
}
void thread_3()
{
    while(flag.load(std::memory_order_acquire) < 2);
    assert(data.at(0) == 42);
    assert(num == 22);
}
int main()
{
    for (int i = 0; i < 100; ++i)
    {
        std::thread a(thread_1);
        std::thread b(thread_2);
        std::thread c(thread_3);
        a.join();
        b.join();
        c.join();
    }
}
```
#### memory_order_seq_cst
当使用该内存序时，同为 memory_order_seq_cst 内存序的变量会以一个简单序列运行，且变量前后所有关于内存的操作都不会被优化重排到该变量之前或之后
最主要的是：变量A如果是在变量B之前执行，那么变量A的前后内存操作语句都会在B之前执行。（memory_order_acq_rel只保证相同变量间的关系）
```cpp
#include <thread>
#include <atomic>
#include <assert.h>
#include <vector>
std::atomic<bool> x{false};
std::atomic<bool> y{false};
std::atomic<int> z{0};
void write_x()
{
    x.store(true, std::memory_order_seq_cst);
}
void write_y()
{
    y.store(true, std::memory_order_seq_cst);
}
void read_x_then_y()
{
    while (!x.load(std::memory_order_seq_cst)); // 如果不使用 memory_order_seq_cst，即无法保证 y.load 在  while (!y.    ::memory_order_seq_cst));之前执行，
    // 那么就可能出现，线程c、d同时把x，y都读取到寄存器，然后同时执行完while语句之后，根据自身寄存器状态，发现x/y是0,因此最终z = 0
    if (y.load(std::memory_order_seq_cst))
    {
        ++z;
    }
}
void read_y_then_x()
{
    while (!y.load(std::memory_order_seq_cst));
    if (x.load(std::memory_order_seq_cst))
    {
        ++z;
    }
}
int main()
{
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join();
    b.join();
    c.join();
    d.join();
    assert(z.load() != 0);
}
```
