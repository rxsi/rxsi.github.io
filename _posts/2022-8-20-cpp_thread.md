---
layout: post
title: 线程
date: 2022-8-20 20:33:04 +0800
categories: C++
tags: C++ 线程同步 线程局部存储
author: Rxsi
---

* content
{:toc}

# 线程
在 linux 系统底层实现中，线程是与其他进程共享数据的进程，也是由`task_struct`结构体表示，因此在 linux 系统线程又被称为轻量级进程。
## 线程的大小
在 linux 中，一个进程/线程能够拥有的栈空间的大小是由系统参数设定的。通过`ulimit -s`命令可知，系统给线程（一个进程中的多个线程有独立的栈空间，而共享堆空间）分配的栈空间的最大大小为`8MB`，**注意系统分配的是虚拟内存，而实际内存只有在被占用时才会因为缺页中断而发生分配**。
```shell
rxsi@VM-20-9-debian:~$ ulimit -s
8192
```
## 一个进程最多可以创建多少个线程
已知 linux 系统会为每一个线程分配`8MB`的虚拟内存，而 32 位系统中，系统最大内存是 4GB，用户区被分配了 3GB 空间，因此理论上可以创建的线程总数为`3GB / 8MB ≈ 300`。而在 64 位系统中，系统最大内存为 2<sup>64</sup>，用户区占用了 128TB，所以理论上可以创建的线程数是无限的（1000多万），不过我们实际装机的物理内存一般也就是几十G，所以首先物理内存方面就不支持创建那么多的线程。

线程的创建实际还受到几个内核参数的限制：
```shell
rxsi@VM-20-9-debian:~$ cat /proc/sys/kernel/threads-max
30295 // 表示系统最大支持的线程数
rxsi@VM-20-9-debian:~$ cat /proc/sys/kernel/pid_max 
32768 // 全局pid数值，因为需要给每个进程/线程分配ID，所以用完的话就会分配失败。僵尸进程会占用该值，因此要避免僵尸进程的产生
rxsi@VM-20-9-debian:~$ cat /proc/sys/vm/max_map_count 
65530 // 一个进程可以拥有的VMA的数量
```
## 内核线程和用户线程
linux 系统有内核线程和用户线程两种类型的线程，内核线程是能够被内核调度和执行的实际线程，因此用户线程需要通过某种关系关联到内核线程上，才能间接被内核进行调度执行。内核线程是由内核创建的线程，通过`ps -AL`列出所有进程，以 k 开头而以 d 结尾的线程一般都为内核线程，常见的有 kthreadd、kswapd 等
```shell
rxsi@VM-20-9-debian:~$ ps -AL
  PID   LWP TTY          TIME CMD
    1     1 ?        00:02:41 systemd // 所有进程的祖先，pid为1
    2     2 ?        00:00:01 kthreadd // 内核守护线程
```
用户线程可以运行在用户态和内核态，而内核线程只能运行于内核态，因此内核线程没有用户空间，即`task_struct`中的`mm_struct`属性是 NULL，当被调度执行时会从前一个被调用的线程的 mm 赋值给后一个内核线程，详见`多进程切换底层实现`TODO。
### 三种线程模型
用户线程与内核线程存在三种线程映射模型，分别为：
* 一对一模型
* 多对一模型
* 多对多模型

#### 一对一模型
用户线程与 KSE 是 **1对1** 关系 (1:1)。大部分编程语言的线程库(如 linux 的 pthread，Java 的 java.lang.Thread，C++11 的 std::thread 等等)都是对操作系统的线程（内核级线程）的一层封装，创建出来的每个线程与一个不同的 KSE 静态关联，因此其调度完全由 OS 调度器来做。这种方式实现简单，直接借助 OS 提供的线程能力，并且不同用户线程之间一般也不会相互影响。但其创建，销毁以及多个线程之间的上下文切换等操作都是直接由 OS 层面亲自来做，在需要使用大量线程的场景下对 OS 的性能影响会很大。
#### 多对一模型
用户线程与 KSE 是 **多对1** 关系 (M:1)，这种线程的创建，销毁以及多个线程之间的协调等操作都是由用户自己实现的线程库来负责，对 OS 内核透明，一个进程中所有创建的线程都与同一个 KSE 在运行时动态关联。现在有许多语言实现的**协程**基本上都属于这种方式。这种实现方式相比内核级线程可以做的很轻量级，对系统资源的消耗会小很多，因此可以创建的数量与上下文切换所花费的代价也会小得多。但该模型有个致命的缺点，如果我们在某个用户线程上调用阻塞式系统调用(如用阻塞方式 read 网络IO)，那么一旦 KSE 因阻塞被内核调度出 CPU 的话，剩下的所有对应的用户线程全都会变为阻塞状态（整个进程挂起）。

所以这些语言的协程库会把自己一些阻塞的操作重新封装为完全的非阻塞形式，然后在以前要阻塞的点上，主动让出自己，并通过某种方式通知或唤醒其他待执行的用户线程在该 KSE 上运行，从而避免了内核调度器由于 KSE 阻塞而做上下文切换，这样整个进程也不会被阻塞了。
#### 多对多模型
用户线程与 KSE 是多对多关系(M:N), 这种实现综合了前两种模型的优点，为一个进程中创建多个 KSE，并且线程可以与不同的 KSE 在运行时进行动态关联，当某个 KSE 由于其上工作的线程的阻塞操作被内核调度出 CPU 时，当前与其关联的其余用户线程可以重新与其他 KSE 建立关联关系。当然这种动态关联机制的实现很复杂，也需要用户自己去实现，这算是它的一个缺点吧。Go 语言中的并发就是使用的这种实现方式，Go 为了实现该模型自己实现了一个运行时调度器来负责 Go 中的"线程"与KSE的动态关联。此模型有时也被称为 **两级线程模型**，**即用户调度器实现用户线程到 KSE 的“调度”，内核调度器实现 KSE 到CPU上的调度**。
## 线程的私有与共享
同一个进程中的线程分别具有私有和共享的属性，归纳如下：
| 线程私有 | 线程共享 |
| --- | --- |
| 
- 局部变量 <br> - 函数的参数<br> - 线程局部变量（TLS）| - 全局/静态变量- 堆上的数据 - 程序代码，任何线程都有权利读取并执行任何代码 - 打开的文件，A线程打开的文件可以由B线程读写 |

## 常见的线程问题
### 1. 主线程退出会使子线程退出吗？
主线程如果在退出时，调用的是`exit`，那么会将整个进程都销毁，因此该进程之下的所有其他线程也会被销毁；但是如果调用的是`pthread_exit`，则只是单纯的退出当前的线程，进程不会受到影响，因此其他线程也可以正常的运行
### 2. 某个线程崩溃了，会导致其他线程或者整个进程退出吗？
线程具有独立的上下文堆栈数据，一个线程奔溃时不会对其他线程造成影响的，但是通常情况下会导致整个进程奔溃（产生 Segment Fault 信号），因此连带着其他线程也就都退出了。当然有些信号是可以捕捉的，如果对这些会造成进程 crash 的信号进行捕捉，那么就可以实现进程的不崩溃了，JVM 就实现了这种机制。
## 线程的基本操作
### 线程创建
#### linux平台
底层内核函数签名如下：
```cpp
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
```

- pthread_t *thread：pthread_t 本质上是一个32位的无符号整数，当创建成功之后，可通过该参数得到对应的线程ID
- const pthread_attr_t *attr：指定了线程的属性，一般为NULL
- void _(_start_routine) (void *)：目标线程函数，这个函数的调用方式要是**cdecl（即C风格的调用类型，默认情况下C/C++都是该调用风格；常见还有**stdcall, __fastcall等） 
   - __cdecl：C/C++默认的缺省方式，函数参数从右至左入栈，且由调用者自行清除这些参数，即生成的二进制可执行代码中都含有对应的清栈代码，因此文件体积会比较大。比方说有一个函数A，那么所有调用A函数的地方都会生成一个相对应的清栈代码
   - __stdcall：函数参数从右至左入栈，被调用函数自行清理内存栈，这样文件体积会小点。
        ```cpp
        void* start_routine(void* args){} // 默认就是__cdecl风格
        // 等价于： 
        void* __cdecl start_routine(void* args){}
        ```
- void *args：传递给线程函数的参数

```cpp
// linux 平台的底层线程函数
// g++ thread_test.cpp -o thread_test -lpthread
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>
#include <iostream>
void threadfunc(void* args)
{
    while(1)
    {
        sleep(1);
        std::cout << "Linux thread func" << std::endl;
        break;
    }
    return NULL;
}

int main()
{
    pthread_t threadid;
    pthread_create(&threadid, NULL, threadfunc, NULL);
    pthread_join(threadid, NULL);
    return 0;
}
```
#### C++11 std::thread类
```cpp
#include <stdio.h>
#include <thread>
#include <iostream>
void threadproc1()
{
    std::cout << "C++ std::thread 1" << std::endl;
}

void threadproc2(int a, int b)
{
    std::cout << "C++ std::thread 2" << std::endl;
}

int main()
{
    std::thread t1(threadproc1); // 这里是普通函数，因此直接使用函数名即可，函数名就是函数指针，如果是类的成员函数那么要用&A::func
    std::thread t2(threadproc2, 1, 2); 
    t1.join(); // 主线程执行到这里会阻塞，直到t1线程执行完毕
    t2.join(); // 主线程执行到这里会阻塞，直到t2线程执行完毕，这样就可以保证main线程退出时t1和t2线程都已经执行完毕
    return 0;
}
```
**在一个函数体含创建的线程要使用**`**join**`**或者**`**detach**`**，否则当函数执行完毕退出后那么线程就会被销毁。**使用`join`时会阻塞直到对应的线程执行完毕才接着执行后面的操作，因此要注意使用的时机，如：
```cpp
void start_fun()
{
    for (int i = 0; i < 5; ++i)
    {
        std::thread t(threadproc1);
        t.join();
    }
}
```
上面在循环中每次都调用了`t.join`，那么实际效果就是后一个线程都会等到前一个线程执行完毕之后才创建执行，因此效率低下。应该修改为以下的做法：
```cpp
void start_fun()
{
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i)
    {
        threads.emplace_back(std::thread(theradproc1));
    }
    for (int i = 0; i < 5; ++i)
    {
        threads[i].join();
    }
}
```
这种方式线程实际就是并发执行，但是函数栈会等待所有线程执行完毕之后才退出

使用`detach`则代表的是把线程独立到后台执行，函数所在线程会失去对该线程的控制权，因此函数栈可以在创建完线程之后就退出，不会影响创建线程的执行。但是这种方式要避免使用，因为如果函数栈有传递局部变量到对应创建的线程中，那么当函数栈退出之后，该变量会被销毁，当线程访问时将会造成不可预期的后果。
```cpp
void start_fun()
{
    for (int i = 0; i < 5; ++i)
    {
        std::thread t(threadproc1); // 没有传递局部变量，因此这样写没问题
        t.detach();
    }
}

void start_fun()
{
    for (int i = 0; i < 5; ++i)
    {
        std::thread t(threadproc1, i); // 传递了局部变量，会造成异常！
        t.detach();
    }
}

```
### 线程ID
#### linux平台

- **pthread_create 的 pthread_t 参数**
- **pthread_t tid = pthread_self();**
- **int tid = syscall(SYS_gettid); // LWP ID**

```cpp
#include <sys/syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <pthread.h>
void* thread_proc(void* args)
{
    pthread_t* tid1 = (pthread_t*) args; // 输出指向线程的内存地址，不是全系统唯一的，因为可能不同进程共享了同一块内存
    int tid2 = syscall(SYS_gettid); // 全系统唯一，是LWP的ID（轻量级进程，早期linux系统的线程是通过进程实现的）
    pthread_t tid3 = pthread_self(); // 输出指向线程的内存地址，不是全系统唯一的，因为可能不同进程共享了同一块内存
}

int main()
{
    pthread_t tid;
    pthread_create(&tid, NULL, thread_proc, &tid);
    pthread_join(tid, NULL);
    return 0;
}
```
#### C++11获取线程ID

- **std::this_thread::get_id()**
- **std::thread.get_id()**

返回的是 **std::thread::id** 类型，需要特殊的转换才能转换为int类型，返回的是LWP ID

### C++使用linux/windows线程函数签名时的注意事项
linux/windows 的线程函数签名有固定要求，以 linux 的线程函数签名为例：
```cpp
void* threadFunc(void* arg);
```
对于类的静态方法，C++编译器都会把这些函数“翻译”为全局函数，即去掉了类的作用域限制。对于类的实例函数，会默认添加类对象实例指针（this指针），即最终函数的效果为：
```cpp
void* threadFunc(Obj* this, void* arg);
```
不符合linux函数签名，因此**只能把线程函数定义为静态方法**
### C++ std::thread类没有函数签名的限制
**对于实例方法，需要添加this指针，或者使用std::bind()函数**
对于静态方法，则不需要
```cpp
class Thread
{
public:
    Thread(){}
    ~Thread(){}
    void Start()
    {
        m_stopped = false;
        m_spThread.reset(new std::thread(&Thread::threadFunc, this, 1, 2)); // 注意,因为调用的是实例对象,因此要主动添加this指针作为参数，这里使用的是类成员函数指针，因此需要使用&Thread::threadFunc
    }
    
    void Stop()
    {
        m_stopped = true;
        if (m_spThread)
        {
            if (m_spThread->joinable())
            {
                m_spThread->join();
            }
        }
    }
    
private:
    void threadFunc(int arg1, int arg2)
    {
        while (!m_stopped)
        {
            std::cout << "Thread function use instance method." << std::endl;
        }
    }
    
    std::shared_ptr<std::thread> m_spThread;
    bool m_stopped;
};

int main()
{
    Thread myThread;
    myThread.Start();
}
```
## 线程同步
### linux互斥体
#### 初始化
```cpp
#include <pthread.h>
// 方式1
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER; // 默认属性，这种方式不需要手动销毁

// 方式2
int pthread_mutex_init(pthread_mutex_t* restrict mutex, const pthread_mutexattr_t* restrict attr); // 可以指定属性，一般属性直接使用 NULL 即可，即默认属性
// restrict 是C的一个关键字，用以修饰后面的指针是指向的变量在本作用域中的唯一指向，当然这只是建议而已~
// 与 restrict 相似的还有 register 关键字，修饰的是变量，建议可以直接从寄存器读取
```
#### 属性设置
```cpp
int pthread_mutexattr_init(pthread_mutexattr_t* attr); // 创建属性对象
int pthread_mutexattr_destory(pthread_mutexattr_t* attr); // 当设置属性之后需要进行销毁
int pthread_mutexattr_settype(pthread_mutexattr_t* attr, int type); // 设置属性
int pthread_mutexattr_gettype(const pthread_mutexattr_t* restrict attr, int* restrict type); // 获取属性
```
可选属性有：

1. `PTHREAD_MUTEX_NORMAL`

普通锁，不同线程只有该锁被释放之后才能再被获取
**如果同一个线程连续加锁会导致阻塞在第二个加锁语句，而如果使用的是 trylock，则返回 EBUSY 错误码**

2. `PTHREAD_MUTEX_ERRORCHECK`

检错锁，不同线程只有该锁被释放之后才能再被获取
**如果同一个线程连续加锁会返回 EDEADLK 错误码**

3. `PTHREAD_MUTEX_RECURSIVE`

可重入锁，不同线程必须等锁被释放之后才能再次加锁
**同一个线程重复加锁会使锁的引用计数+1，每次调用解锁则是引用计数-1，因此需要成对出现**
#### 销毁
```cpp
int pthread_mutex_destory(pthread_mutex_t* mutex); // 执行成功会返回0
```
需要注意两点：

1. 无须销毁 **PTHREAD_MUTEX_INITIALIZER **初始化的互斥体;
2. 如果主动调用`pthread_mutex_destory`销毁一个正在被使用的锁（包括应用在`pthread_cond_wait`），那么会返回`EBUSY`错误码。而如果正在被使用的`mutex`对象被销毁了，那么会造成不可预测的后果，例如线程是以`detach`的方式运行而保存了主线程的局部`mutex`对象引用
#### 加锁解锁
```cpp
int pthread_mutex_lock(pthread_mutex_t* mutex); // 尝试加锁，如果失败会一直阻塞
int pthread_mutex_trylock(pthread_mutex_t* mutex); // 尝试加锁，如果失败会立即放回
int pthread_mutex_unlock(pthread_mutex_t* mutex);

// 加锁成功后返回0
```
#### 示例代码
```cpp
#include <pthread.h>
#include <stdio.h>
#include <errno.h>
#include <iostream>
#include <unistd.h>

pthread_mutex_t mymutex;

void* thread_fun(void* args)
{
    pthread_mutex_lock(&mymutex);
    pthread_t* threadID = (pthread_t*) args;
    std::cout << threadID << ": thread running" << std::endl;
    sleep(3);
    
    // 尝试销毁被锁定的mutex对象
    int  ret = pthread_mutex_destroy(&mymutex);
    if (ret != 0)
    {
        if (ret == EBUSY)
        {
            std::cout << "EBUSY" << std::endl;
            std::cout << "Failed to destory mutex." << std::endl;
        }
    }
    pthread_mutex_unlock(&mymutex);
    return NULL;
}

int main()
{
    pthread_mutex_init(&mymutex, NULL); // 一般不用检查初始化结果
    pthread_t threadID1, threadID2, threadID3;
    pthread_create(&threadID1, NULL, thread_fun, &threadID1);
    pthread_create(&threadID2, NULL, thread_fun, &threadID2);
    pthread_create(&threadID3, NULL, thread_fun, &threadID3);
    pthread_join(threadID1, NULL);
    pthread_join(threadID2, NULL);
    pthread_join(threadID3, NULL);
    // 尝试销毁已经解锁的mutex对象
    int ret = pthread_mutex_destroy(&mymutex);
    if (ret == 0)
    {
        std::cout << "Successed to destory mutex." << std::endl;
    }
    return 0;
}
```
### linux信号量
这个信号又称为 Posix 的无名信号量，是基于内存的。而有名信号量基于文件系统提供了标识，底层本质还是基于内存。
各信号量区别如下：

| 功能 | 有名信号量 | 无名信号量 | System V信号量 |
| --- | --- | --- | --- |
| 头文件 | semaphore.h | semaphore.h | sys/sem.h |
| 初始化 | sem_t *sem_open(const char *name, int oflag, ...);
/* mode_t mode, unsigned int value*/ | int sem_init(sem_t* sem, int pshared, unsigned int value); | int semget(key_t key, int nsems, int oflag); |
|  | 参数：
const char *name：系统真实路径，**以单斜杠开头，且不能包含其他的单斜杠**

int oflag：可以是0、O_CREATE 或 O_CREATE&#124;O_EXCL，O_EXCL 表示创建时该信号量必须是不存在的，使用 0 则单纯表示打开已有的信号量

mode_t mode：指定权限位，只有当指定了O_CREATE标志才需要

unsigned int value：初始的计数量，不能超过SEM_VALUES_MAX，只有当指定了 O_CREATE 标志才需要 | 参数：
sem_t *sem：自定义的信号指针，当传入经过初始化后，该指针指向内核创建的信号量

int pshared：0 表示信号量是在同一个进程中的各个线程共享的，否则该信号量是在进程间共享的。当为进程间共享时，需要放在某种类型的共享内存中

unsigned int value：代表初始时的资源量
 | 参数：
key_t key：信号量key

int nsems：初始信号量数，如果只是访问一个已存在的集合，则指定为0即可

int oflag：SEM_R（读）、SEM_A（改），还可以与IPC_CREAT 或 IPC_CREAT &#124; IPC_EXCL按位或 |
|  | 返回：
成功返回指向信号量的指针，出错返回SEM_FAILED | 返回：
出错返回-1 | 返回：
成功返回非负标识符，出错返回-1 |
|  | 说明：
用以创建一个新的有名信号量或者打开一个已经存在的有名信号量，可用于进程/线程间同步 | 说明：
基于内存的信号量，具体的持续性取决于底层的内存类型，如共享内存类型的会一直持续直到主动释放 | 说明：
当创建信号集之后， 内核会维护`semid_ds`结构体（可以同时包含多个信号量），存储信号量的访问权限、信号值等。
 |
| 信号量减少 | int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem); |  | int semop(int semid, struct sembuf *opsptr, size_t nops);

 |
|  | 参数：
sem_t *sem：信号量指针 |  |  |
|  | 返回：
成功返回0，出错返回-1 |  | 参数：
struct sembuf *opsptr：指向`sembuf`数组

size_t nops：表明opsptr所指向的数组的大小 |
|  | 说明：
当信号量大于0时，调用上述方法会使信号量减一，然后返回；如果信号量为0，`sem_wait`会阻塞等待，而`sem_trywait`会立即返回，且`errno=EAGAIN`。阻塞等待的过程中可能会被其他信号中断，且`errno=EINTR`，所以要用`while`循环等待 |  |  |
| 信号量增加 | int sem_post(sem_t *sem); |  | 返回：
成功返回0，出错返回-1 |
|  | 参数：
sem_t *sem：信号量指针 |  |  |
|  | 返回：
成功返回0，出错返回-1 |  | 说明：
用来对信号集执行`PV`操作
P：申请资源，信号-1
V：释放资源，信号+1 |
|  | 说明：
当一个线程/进程用完当前的信号后，要调用`sem_post`方法，使信号量+1，然后就会唤醒其他正在等待信号的其他线程/进程 |  |  |
| 获取当前信号量 | int sem_getvalue(sem_t *sem, int *valp); |  | int semctl(int semid, int semmum, int cmd, ... /* union semun arg*/); |
|  | 参数：
sem_t *sem：信号量指针

int *valp：整数指针，用以存储当前的信号量值 |  |  |
|  | 返回：
成功返回0，失败返回-1 |  | 参数：
int semid：非负的信号标识符

int semmum：信号量集内的某个成员（0、1...nsems-1），只有当cmd为`GETVAL、SETVAL、GETNCNT、GETZCNT、GETPID`时有效

int cmd：指定了semctl的行为，常见项有：
`GETVAL、SETVAL、GETNCNT、GETZCNT、GETPID`等

union semun arg：可选参数，取决于cmd类型，该结构体需要程序自定义 |
|  | 说明：
如果当前信号量已经上锁，则返回值为0，或为某个负数，其绝对值就是等待该信号量解锁的线程数 |  |  |
| 关闭 | int sem_close(sem_t *sem);
int sem_unlink(const char *name); | int sem_destory(sem_t *sem); | 返回：
成功返回非负数，出错返回-1 |
|  | 参数：
sem_t *sem：信号量指针
const char *name：路径名 | 参数：
sem_t *sem：信号量指针 |  |
|  | 返回：
成功返回0，出错返回-1 | 返回：
成功返回0，出错返回-1 | 说明：
用来对单个信号或者整个信号集执行操作 |
|  | 说明：
当一个进程终止时，内核会对其打开着的有名信号量自动执行`sem_close`方法，但是只有当执行了`sem_unlink`时才会真正删除，且当该有名信号量的引用计数大于0时，不会调用析构。 | 说明：
释放信号量 |  |

#### 初始化
```cpp
#include <semaphore.h>
int sem_init(sem_t* sem, int pshared, unsigned int value); 
// pshared代表的是是否可以被不同进程共享，0否，非0是
// value代表初始时的资源量
```
#### 销毁
```cpp
int sem_destory(sem_t* sem);
```
#### 消耗量资源增加
```cpp
int sem_post(sem_t* sem);
// 使信号量的资源数+1,且唤醒所有因sem_wait函数被阻塞的其他线程都会被唤醒
```
#### 消耗量资源减少
```cpp
int sem_wait(sem_t* sem);
// 当信号量为0时会阻塞，当信号量大于0时会被唤醒，唤醒后会把资源计数-1,然后立即返回执行后面的操作

int sem_trywait(sem_t* sem);
// 当信号量为0时会立即返回，返回值是-1,错误码errno == EAGAIN

int sem_timedwait(sem_t* sem, const struct timespec* abs_timeout);
// 带有等待时间的等待，超时后返回-1,错误码 errno == ETIMEDOUT


// 三个wait函数都会被系统中断，会立即返回，返回值-1,错误码 errno == EINTR

struct timespec
{
    time_t tv_sec; 秒
    long tv_nsec; 纳秒
}
// 注意temespec是绝对时间
```
#### 示例代码
```cpp
#include <pthread.h>
#include <errno.h>
#include <unistd.h>
#include <list>
#include <semaphore.h>
#include <iostream>

class Task
{
public:
    Task(int myTaskID): taskID(myTaskID){}

    void doTask()
    {
        std::cout << "handle a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
    }

private:
    int taskID;
};

pthread_mutex_t mymutex;
std::list<Task*> tasks;
sem_t mysemapthore;

void* consumer_thread(void* param)
{
    Task* pTask = NULL;
    while (true)
    {
        if (sem_wait(&mysemapthore) != 0) // 本身方法是阻塞的，但是有可能会被系统中断，因此需要判断是否为0，且外层使用loop循环尝试
            continue;
        if (tasks.empty())
            continue;

        pthread_mutex_lock(&mymutex);
        pTask = tasks.front();
        tasks.pop_front();
        pthread_mutex_unlock(&mymutex);

        pTask->doTask();
        delete pTask;
    }
    return NULL;
}

void* producer_thread(void* param)
{
    int taskID = 0;
    Task* pTask = NULL;

    while (true)
    {
        pthread_mutex_lock(&mymutex);
        pTask = new Task(taskID);
        tasks.push_back(pTask);
        std::cout << "produce a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
        pthread_mutex_unlock(&mymutex);

        // 释放信号量，通知消费者
        sem_post(&mysemapthore);
        taskID++;
        sleep(1);
    }
    return NULL;
}

int main()
{
    pthread_mutex_init(&mymutex, NULL); // 用以保证tasks列表操作的安全性
    sem_init(&mysemapthore, 0, 0);

    // 五个消费线程
    pthread_t consumerThreadID[5];
    for (int i = 0; i < 5; i++)
        pthread_create(&consumerThreadID[i], NULL, consumer_thread, NULL);

    // 一个生产者线程
    pthread_t producerThreadID;
    pthread_create(&producerThreadID, NULL, producer_thread, NULL);

    pthread_join(producerThreadID, NULL);

    for (int i = 0; i < 5; ++i)
        pthread_join(consumerThreadID[i], NULL);

    sem_destroy(&mysemapthore);
    pthread_mutex_destroy(&mymutex);

    return 0;
}
```
### linux条件变量
#### 初始化
```cpp
int pthread_cond_init(pthread_cond_t* cond, const pthread_condattr_t* attr);
或者：
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```
#### 销毁
```cpp
int pthread_cond_destory(pthread_cond_t* cond);
```
#### 等待
```cpp
int pthread_cond_wait(pthread_cond_t* restrict cond, pthread_mutex_t* restrict mutex);
int pthread_cond_timewait(pthread_cond_t* restrict cond, pthred_mutex_t* restrict mutex, const struct timespec* restrict abstime);
// 条件变量的方法需要同时传入互斥体，因为信号变量必须在信号发出时，线程已经有调用wait操作，因此需要把wait和互斥体的unlock变成一个原子操作。否则可能出现互斥体刚unlock，然后失去了时间片，而另外的线程发出信号，切换回本线程之后已经失去了信号了，造成永久阻塞
// 当条件未满足时，线程会一直等待
// 注意temespec是绝对时间
```
#### 唤醒
```cpp
int pthread_cond_signal(pthread_cond_t* cond);  // 一次唤醒一个线程
int pthread_cond_broadcast(pthread_cond_t* cond); // 同时唤醒所有的线程
```
#### 为什么需要配合 mutex
以下面两个线程的执行为例，我们的本意是**避免线程进行永久阻塞状态**
```cpp
// threadA
pthread_mutex_lock(&mutex);
while (!ready) // 这里的条件判断和pthread_cond_wait通过mutex形成了原子性语句
    pthread_cond_wait(&cond, &mutex);
pthread_mutex_unlocK(&mutex);

// threadB
pthread_mutex_lock(&mutex);
ready = true;
pthread_mutex_unlock(&mutex);
pthread_cond_signal(&cond);
```
使用`mutex`的唯一目的是避免 threadA 在判断 ready 为 false 之后，刚准备调用`pthread_cond_wait`函数，而失去了时间片，然后 threadB 修改了 ready 的值，并调用`pthread_cond_signal`。如果上述情况发生，那么 threadA 就会永远进行阻塞状态，因此 threadA 在判断 ready 值时需要先获得锁，以保证在进行阻塞状态之前该变量不会被其他线程修改。
而在 threadA 进入阻塞状态之前是需要 unlock 锁的，此时如果没有设计为原子操作，则可能在 unlock 时，刚好就失去了时间片，而后 threadB 进行了修改，threadA 再进入了阻塞状态，就再也无法被唤醒了。
当然如果上面的 threadB 终究是先于 threadA 执行，那么对于 threadA 来说，此时的 ready 值为 true，因此不会再调用`pthread_cond_wait`函数，也就不会进入阻塞状态了，不过这个信号也就丢失了
#### 示例代码
```cpp
#include <pthread.h>
#include <errno.h>
#include <unistd.h>
#include <list>
#include <semaphore.h>
#include <iostream>

class Task
{
public:
    Task(int myTaskID): taskID(myTaskID){}

    void doTask()
    {
        std::cout << "handle a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
    }

private:
    int taskID;
};

pthread_mutex_t mymutex;
std::list<Task*> tasks;
pthread_cond_t mycv;

void* consumer_thread(void* param)
{
    Task* pTask = NULL;
    while (true)
    {
        pthread_mutex_lock(&mymutex);

        // 这里使用while的原因是可能会被虚假唤醒，因此当wait被唤醒之后，还需要判断tasks是否为空
        while (tasks.empty())
            // 如果获得了互斥锁，但是条件不合适，则pthread_cond_wait 会释放锁，不往下执行，所以在这语句之前有lock的操作
            // 发生变化后，如果条件合适，则pthread_cond_wait会阻塞到获得锁，可能此时有多个线程都进入到这里，在 wait 条件的达成，而且 post 可能会同时唤醒所有线程，但是这些线程只会有一个线程拿到这个互斥体锁，也就只有一个线程会得到运行的机会。由于有加锁的动作，因此在这语句之后有unlock的操作
            pthread_cond_wait(&mycv, &mymutex);


        pTask = tasks.front();
        tasks.pop_front();

        pthread_mutex_unlock(&mymutex);

        if (pTask == NULL)
            continue;

        pTask->doTask();
        delete pTask;
        pTask = NULL;
    }

    return NULL;
}

void* producer_thread(void* param)
{
    int taskID = 0;
    Task* pTask = NULL;

    while (true)
    {
        pTask = new Task(taskID);
        pthread_mutex_lock(&mymutex);
        tasks.push_back(pTask);
        std::cout << "produce a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
        pthread_mutex_unlock(&mymutex);

        pthread_cond_signal(&mycv);
        taskID++;
        sleep(1);
    }
    return NULL;
}

int main()
{
    pthread_mutex_init(&mymutex, NULL); // 用以保证tasks列表操作的安全性
    pthread_cond_init(&mycv, NULL);

    // 五个消费线程
    pthread_t consumerThreadID[5];
    for (int i = 0; i < 5; i++)
        pthread_create(&consumerThreadID[i], NULL, consumer_thread, NULL);

    // 一个生产者线程
    pthread_t producerThreadID;
    pthread_create(&producerThreadID, NULL, producer_thread, NULL);

    pthread_join(producerThreadID, NULL);

    for (int i = 0; i < 5; ++i)
        pthread_join(consumerThreadID[i], NULL);

    pthread_cond_destroy(&mycv);
    pthread_mutex_destroy(&mymutex);

    return 0;
}
```
### linux读写锁
读写锁和互斥体的差别在于：
读锁和读锁是共享的，而写锁和写锁、写锁和读锁都是互斥的
#### 初始化
```cpp
int pthread_rwlock_init(pthread_rwlock_t* rwlock, const pthread_rwlockattr_t* attr); // 属性一般为NULL即可
或者:
pthread_rwlock_t myrwlock = PTHREAD_RWLOCK_INITIALIZED;
```
#### 属性设置
```cpp
int pthread_rwlockattr_setkind_np(pthread_rwlockattr_t* attr, int pref);
int pthread_rwlockattr_getkind_np(const pthread_rwlockattr_t* attr, int* pref);

// pref参数用于设置读写锁的类型，取值如下：
enum
{
    // 读者优先
    PTHREAD_RWLOCK_PREFER_READER_NP,
    PTHREAD_RWLOCK_PREFER_WRITER_NP,

    // 写者优先
    PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP,

    PTHREAD_RWLOCK_DEFAULT_NP = PTHREAD_RWLOCK_PREFER_READER_NP （默认属性，依然是读者优先）
}

// 属性初始化和销毁
int pthread_rwlockattr_init(pthread_rwlockattr_t* attr);
int pthread_rwlockattr_destory(pthread_rwlockattr_t* attr);
```
#### 销毁
```cpp
int pthread_rwlock_destory(pthread_rwlock_t* rwlock);
```
#### 请求读锁
```cpp
int pthread_rwlock_rdlock(pthread_rwlock_t* rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t* rwlock); // 当不能获取锁时返回-1,errno == EBUSY
int pthread_rwlock_timedrdlock(pthread_rwlock_t* rwlock, const struct timespec* abstime); // 超时后返回 -1 && errno == ETIMEOUT
```
#### 请求写锁
```cpp
int pthread_rwlock_wrlock(pthread_rwlock_t* rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t* rwlock); // 当不能获取锁时返回-1,errno == EBUSY
int pthread_rwlock_timedwrlock(pthread_rwlock_t* rwlock, const struct timespec* abstime); //超时后返回 -1 && errno == ETIMEOUT
```
#### 释放锁
```cpp
int pthread_rwlock_unlock(pthread_rwlock_t* rwlock);
```
#### 示例代码
```cpp
#include <pthread.h>
#include <unistd.h>
#include <iostream>

int resourceID = 0;
pthread_rwlock_t myrwlock;

void* read_thread(void* param)
{
    while (true)
    {
        pthread_rwlock_rdlock(&myrwlock);
        std::cout << "read thread ID: " << pthread_self() << ", resourceID: " << resourceID << std::endl;

        sleep(1);
        pthread_rwlock_unlock(&myrwlock);
    }
    return NULL;
}

void* write_thread(void* parm)
{
    while (true)
    {
        pthread_rwlock_wrlock(&myrwlock);

        ++resourceID;
        std::cout << "write thread ID: " << pthread_self() << ", resourceID: " << resourceID << std::endl;

        sleep(1);
        pthread_rwlock_unlock(&myrwlock);
    }

    return NULL;
}

int main()
{
    pthread_rwlock_init(&myrwlock, NULL); // 由于默认是读优先，因此写线程很难获得机会执行
    pthread_t readThreadID[5];
    for (int i = 0; i < 5; ++i)
        pthread_create(&readThreadID[i], NULL, read_thread, NULL);

    pthread_t writeThreadID;
    pthread_create(&writeThreadID, NULL, write_thread, NULL);
    pthread_join(writeThreadID, NULL);
    for (int i = 0; i < 5; ++i)
        pthread_join(readThreadID[i], NULL);

    pthread_rwlock_destroy(&myrwlock);
    return 0;
}
```
### std::mutex互斥体
windows 平台和 linux 平台通用
正常的做法是 lock 和 unlock 成对出现，为了避免出现差错，因此使用 RAII 封装了几个接口，类似于智能指针的做法
```cpp
lock_guard      // 基于作用域的互斥体管理,即使用{}划定范围,当调用时会自动调用 std::mutex 的lock方法,当出了作用域时会自动调用 std::mutex 的unlock方法
// 示例：
// sd::mutex m;
// std::lock_guard<std::mutex> guard(m);
        
unique_lock     // 灵活的互斥体管理,可以转让控制权等，用在条件变量的wait时，因为锁要在wait内部释放，所以需要转让管理权。同时读写锁的写锁也是通过他管理
// 示例：
// std::mutex m;
// std::unique_lock<std::mutex> unique(m);
// std::shared_mutex m2;
// std::unique_lock<std::shared_mutex> unique2(m2);

shared_lock     // 共享互斥体管理，只适用于读写锁里面的读锁
// 示例：
// std::shared_mutex m;
// std::shared_lock<std::shared_mutex> shared(m);

scoped_lock     // 多互斥体避免死锁管理，可以同时申请多个互斥体，并且不会造成死锁
// 示例：
// std::mutex m1, m2;
// std::scoped_lock<std::mutex, std::mutex> scoped(m1, m2);

void fun()
{
    std::mutex m; // 这种方法是错的,因为std::mutex的作用域一定要超过fun()的作用域
    std::lock_guard<std::mutex> guard(m);
}
```
#### 示例代码
```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <mutex>
#include <atomic>

int g_num = 0;
std::mutex g_num_mutex;

void slow_increment(int id)
{
    for (int i = 0; i < 3; ++i)
    {
        g_num_mutex.lock(); // lock要和unlock配对使用，否则如果连续重复加锁会导致阻塞
        ++g_num;
        std::cout << id << " => " << g_num << std::endl;
        g_num_mutex.unlock();

        std::this_thread::sleep_for(std::chrono::seconds(1)); // 休眠1秒
    }
}

int main()
{
    std::thread t1(slow_increment, 0);
    std::thread t2(slow_increment, 1);
    t1.join();
    t2.join();
    return 0;
}
```
### std::shared_mutex读写锁
底层是操作系统提供的读写锁
```cpp
lock / unlock // 用来获取写锁和解除写锁(排他锁)
lock_shared / unlock_shared // 用来获取读锁和解除读锁(共享锁)
unique_lock(写锁) 和 shared_lock(读锁) // 用来以RAII方式自动对std::shared_mutex加锁和解锁
```
#### 示例代码
```cpp
#define READER_THREAD_COUNT 8
#define LOOP_COUNT 5000000

#include <iostream>
#include <mutex> // 互斥体
#include <shared_mutex> // 共享互斥体
#include <thread>

class shared_mutex_counter
{
public:
    shared_mutex_counter() = default;
    ~shared_mutex_counter() = default;

    unsigned int get() const
    {
        std::shared_lock<std::shared_mutex> lock(m_mutex); // 共享锁，退出函数之后会自动释放锁
        return m_value;
    }

    void increment()
    {
        std::unique_lock<std::shared_mutex> lock(m_mutex); // 排他锁
        m_value++;
    }

    void reset()
    {
        std::unique_lock<std::shared_mutex> lock(m_mutex);
        m_value = 0;
    }

private:
    mutable std::shared_mutex m_mutex;
    unsigned int m_value = 0;
};                                       

class mutex_counter
{
public:
    mutex_counter() = default;
    ~mutex_counter() = default;

    unsigned int get() const
    {
        std::unique_lock<std::mutex> lock(m_mutex); // 互斥体,相当于排他锁，当退出该函数后会自动释放锁
        return m_value;
    }

    void increment()
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        m_value++;
    }

    void reset()
    {
        std::unique_lock<std::mutex> lock(m_mutex);
        m_value = 0;
    }

private:
    mutable std::mutex m_mutex;
    unsigned int m_value = 0;
};

void test_shared_mutex()
{
    shared_mutex_counter counter;
    int temp;

    // 写线程函数
    auto writer = [&counter]() {
        for (int i = 0; i < LOOP_COUNT; i++)
            counter.increment();
    };

    // 读线程函数
    auto reader = [&counter, &temp]() {
        for (int i = 0; i < LOOP_COUNT; i++)
            temp = counter.get();
    };

    // 存放读线程对象指针的数组
    std::thread** tarray = new std::thread* [READER_THREAD_COUNT];
    
    clock_t start = clock();
    for (int i = 0; i < READER_THREAD_COUNT; i++)
        tarray[i] = new std::thread(reader);
    std::thread tw(writer);
    for (int i = 0; i < READER_THREAD_COUNT; i++)
        tarray[i]->join();
    tw.join();

    clock_t end = clock();
    std::cout << "[test_shared_mutex]" << std::endl;
    std::cout << "thread count: " << READER_THREAD_COUNT << std::endl;
    std::cout << "result: " << counter.get() << ", cost: " << end - start << ", temp: " << temp << std::endl;
}

void test_mutex()
{
    mutex_counter counter;
    int temp;
    auto writer = [&counter]() {
        for (int i = 0; i < LOOP_COUNT; i++)
            counter.increment();
    };

    auto reader = [&counter, &temp]() {
        for (int i = 0; i < LOOP_COUNT; i++)
            temp = counter.get();
    };

    std::thread** tarray = new std::thread* [READER_THREAD_COUNT];
    clock_t start = clock();
    for (int i = 0; i < READER_THREAD_COUNT; i++)
        tarray[i] = new std::thread(reader);
    std::thread tw(writer);
    for (int i = 0; i < READER_THREAD_COUNT; i++)
        tarray[i]->join();
    tw.join();
    clock_t end = clock();
    std::cout << "[test_mutex]" << std::endl;
    std::cout << "thread count: " << READER_THREAD_COUNT << std::endl;
    std::cout << "result: " << counter.get() << ", cost: " << end - start << ", temp: " << temp << std::endl;
}

int main()
{
    test_mutex();
    // test_shared_mutex();
    return 0;
}
```
### std::condition_variable条件变量
#### 示例代码
```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <list>
#include <iostream>

class Task
{
public:
    Task(int myTaskID): taskID(myTaskID){}

    void doTask()
    {
        std::cout << "handle a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
    }

private:
    int taskID;
};

std::mutex mymutex; // C++代码,可以同时兼容linux 和 windows
std::list<Task*> tasks;
std::condition_variable mycv;

void* consumer_thread()
{
    Task* pTask = NULL;
    while (true)
    {
        {
            std::unique_lock<std::mutex> guard(mymutex); // c++的是使用RAII封装,因此使用时需要限定使用范围,以达到退出范围时自动释放的功能
            while (tasks.empty())
            {
                mycv.wait(guard);
            }
            pTask = tasks.front();
            tasks.pop_front();
        }
        if (pTask == NULL)
            continue;

        pTask->doTask();
        delete pTask;
        pTask = NULL;
    }

    return NULL;
}

void* producer_thread()
{
    int taskID = 0;
    Task* pTask = NULL;

    while (true)
    {
        pTask = new Task(taskID);
        { // 使用{}减小作用范围
            std::lock_guard<std::mutex> guard(mymutex); // 注意mymutex的生命周期要大于guard!!!!!!!!!!!!!!
            tasks.push_back(pTask);
            std::cout << "produce a task, taskID: " << taskID << ", threadID: " << pthread_self() << std::endl;
        }
        std::cout << "is empty = " << tasks.empty() << std::endl;
        mycv.notify_one(); // 发出信号
        taskID++;
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
    return NULL;
}

int main()
{
    // 五个消费线程
    std::thread consumerThreadID[5];
    for (int i = 0; i < 5; i++)
        consumerThreadID[i] = std::thread(consumer_thread);

    // 一个生产者线程
    std::thread producer(producer_thread);
    producer.join();
    for (int i = 0; i < 5; ++i)
        consumerThreadID[i].join();
    
    return 0;
}
```
条件变量的应用：确保线程启动成功
```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <iostream>
#include <vector>
#include <memory>

std::mutex mymutex;
std::condition_variable mycv;
bool success = false;


void thread_func(int no)
{
    {
        std::unique_lock<std::mutex> lock(mymutex); 
        success = true; // 逻辑跑到这里说明线程已经被创建成功
        mycv.notify_all();
    }
    std::cout << "thread create success, thread no: " << no << std::endl;
    while (true); // 简单死循环模拟
}

int main()
{
    std::vector<std::shared_ptr<std::thread>> threads;
    for (int i = 0; i < 5; ++i)
    {
        success = false;
        // std::shared_ptr<std::thread> spthread;
        // spthread.reset(new std::thread(thread_func, i));
        auto spthread = std::make_shared<std::thread>(thread_func, i);
        {
            std::unique_lock<std::mutex> lock(mymutex);
            while (!success)
                mycv.wait(lock); // 虽然这里在未接收到信号时会释放锁,但是该锁并不会被本线程获取到,因此本线程会一直阻塞在这里
        }
        std::cout << "start thread success: thread no: " << i << std::endl;
        threads.emplace_back(spthread);
    }

    for (auto& iter: threads)
        iter->join();

    return 0;
}
```
## 线程局部存储（TLS，Thread Local Storage）
使用线程局部存储的目的在于**实现线程独占的全局变量**，因此这是栈和寄存器无法实现的（这两个也是每个线程独占的数据）
### 实现原理
从代码的反汇编入手：
```cpp
// 代码段1：
void thread_func()
{
    int global_g = 1; 	// mov     DWORD PTR [rbp-4], 1
    global_g++;			// add     DWORD PTR [rbp-4], 1
    return;
}
// 代码1中 global_g 是局部变量，因此会往栈中压入1，然后再执行+1操作


// 代码段2：
int global_g = 1;
void thread_func()
{
    global_g++; // mov     eax, DWORD PTR global_g[rip]
                // add     eax, 1
                // mov     DWORD PTR global_g[rip], eax
    return;
}
// 代码2中 global_g 是全局变量，因此是从rip存储的指令地址（指向的是全局变量存储区，地址是在编译之后就固定的）中获取到变量，然后+1之后，再写回

// 代码段3：
thread_local int global_g = 1;
void thread_func()
{
    global_g++; // mov     eax, DWORD PTR fs:global_g@tpoff
                // add     eax, 1
                // mov     DWORD PTR fs:global_g@tpoff, eax
    return;
}
// 代码3中 global_g 是根据 FS 寄存器指向的地址再加上一定的偏移量获取到的
```
从上面演示的代码可以知道`TLS`的获取是根据`FS`寄存器指向的地址再加上一定的偏移量获取的。而实际上`FS`寄存器存储的是当前线程的`TCB`的指针，这个`TCB`块就存储在`task_struct`中的`thread_struct`中的`desc_struct`数组中。
```cpp
struct task_struct {
    //... 忽略代码
    struct thread_struct		thread;
};

struct thread_struct {
	/* Cached TLS descriptors: */
	struct desc_struct	tls_array[GDT_ENTRY_TLS_ENTRIES];
    //... 忽略代码
};
```
`tls_array`数组的容量为3，分别对应了三个`TLS segment`
```cpp
/*
 * The layout of the per-CPU GDT under Linux:
 // ..... 忽略
 *  ------- start of TLS (Thread-Local Storage) segments:
 *
 *   6 - TLS segment #1			[ glibc's TLS segment ] // glibc用这个段存储TLS，fs寄存器指向的就是这个段
 *   7 - TLS segment #2			[ Wine's %fs Win32 segment ]
 *   8 - TLS segment #3							<=== cacheline #3
// .... 忽略
```
linux 系统通过`do_set_thread_area`函数设置`TLS`
```cpp
/*
 * Set a given TLS descriptor:
 */
int do_set_thread_area(struct task_struct *p, int idx,
		       struct user_desc __user *u_info,
		       int can_allocate)
{
    // .... 忽略代码
	set_tls_desc(p, idx, &info, 1); // 
    // .... 忽略代码
}


static void set_tls_desc(struct task_struct *p, int idx,
			 const struct user_desc *info, int n)
{
	struct thread_struct *t = &p->thread;
	struct desc_struct *desc = &t->tls_array[idx - GDT_ENTRY_TLS_MIN];
	int cpu;

	/*
	 * We must not get preempted while modifying the TLS.
	 */
	cpu = get_cpu();

	while (n-- > 0) {
		if (LDT_empty(info) || LDT_zero(info))
			memset(desc, 0, sizeof(*desc));
		else
			fill_ldt(desc, info);
		++info;
		++desc;
	}

	if (t == &current->thread)
		load_TLS(t, cpu); // 将TLS内容载入到CPU中

	put_cpu();
}
```
### linux平台
#### pthread_keys
```cpp
int pthread_key_create(pthread_key_t* key, void (*destructor)(void*)); 
// 函数会调用 pthread_key_create 函数申请一个槽位,返回一个小于1024的无符号整数填入pthread_key_t中, 一共有1024个槽位。记录槽位分配情况的数据结构pthread_keys是进程唯一的

// destructor是自定义函数指针,函数签名如下:
void* destructor(void* value)
{
    // .. 当线程终止时,如果key关联的值不为NULL,则会自动执行该函数,如果无需析构,那么将destructor参数设置为NULL即可
}


int pthread_key_delete(pthread_key_t key); // 删除
int pthread_setspecific(pthread_key_t key, const void* value); // 设置
void* pthread_getspecific(pthread_key_t key); // 获取
```
#### __thread
```cpp
__thread int val = xxx; 
```
### C++11
#### thread_local
```cpp
thread_local int val = xxx;
```