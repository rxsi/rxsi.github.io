---
layout: post
title: 进程宕机原因分析
date: 2022-5-7 10:58:52 +0800
categories: linux
tags: process crash
author: Rxsi
---

* content
{:toc}

## 进程处于S状态
### strace命令
当进程处于`S`状态，说明当前进程正处于**可中断的阻塞状态**，正在阻塞的等待某个函数调用的返回，可以使用`strace`命令，查看进程当前究竟阻塞在哪个函数调用。
以`blocking_server`为例：
```shell
rxsi@VM-20-9-debian:~$ ps -aux | grep blocking_server
rxsi     5777  0.0  0.0   6564  1728 pts/0    S+   16:56   0:00 ./blocking_server
rxsi     5989  0.0  0.0   7256   892 pts/2    S+   16:56   0:00 grep blocking_server
```
`strace`命令查看：
```shell
rxsi@VM-20-9-debian:~$ strace -T -tt -e trace=all -p 5777
strace: Process 5777 attached
15:41:02.639978 accept(3, 
```
可见当前正阻塞在`accept`方法，且当前的套接字`fd == 3`
使用`lsof`可以查看当前进程打开的套接字
```shell
rxsi@VM-20-9-debian:~$ lsof -p 5777
COMMAND    PID USER   FD   TYPE    DEVICE SIZE/OFF    NODE NAME
blocking_ 5777 rxsi  cwd    DIR     254,1     4096 1192109 /home/rxsi/learncpp/socket_test/block_socket
blocking_ 5777 rxsi  rtd    DIR     254,1     4096       2 /
blocking_ 5777 rxsi  txt    REG     254,1    17624 1192141 /home/rxsi/learncpp/socket_test/block_socket/blocking_server
blocking_ 5777 rxsi  mem    REG     254,1    14592  269841 /usr/lib/x86_64-linux-gnu/libdl-2.28.so
blocking_ 5777 rxsi  mem    REG     254,1  1820400  269837 /usr/lib/x86_64-linux-gnu/libc-2.28.so
blocking_ 5777 rxsi  mem    REG     254,1   100712  262180 /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
blocking_ 5777 rxsi  mem    REG     254,1  1579448  269843 /usr/lib/x86_64-linux-gnu/libm-2.28.so
blocking_ 5777 rxsi  mem    REG     254,1  1570256  265583 /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.25
blocking_ 5777 rxsi  mem    REG     254,1    42880  269177 /usr/lib/x86_64-linux-gnu/libonion_security.so.1.0.19
blocking_ 5777 rxsi  mem    REG     254,1   165632  269829 /usr/lib/x86_64-linux-gnu/ld-2.28.so
blocking_ 5777 rxsi    0u   CHR     136,0      0t0       3 /dev/pts/0
blocking_ 5777 rxsi    1u   CHR     136,0      0t0       3 /dev/pts/0
blocking_ 5777 rxsi    2u   CHR     136,0      0t0       3 /dev/pts/0
blocking_ 5777 rxsi    3u  IPv4 168274455      0t0     TCP *:3000 (LISTEN)
```
可以看到 3 号套接字是`TCP`协议，且当前处于`LISTEN`状态
### gdb命令
可以使用`gdb program -p pid`命令连接到正在运行的进程，然后看其当前被挂起的地方
```shell
rxsi@VM-20-9-debian:~$ gdb program -p 5777
```
然后使用`bt`命令查看当前的堆栈信息
```shell
(gdb) bt
#0  0x00007f0481a9fe34 in accept () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x000056226bdab317 in main ()
```
可以看到当前进程被阻塞在`accept`函数
## 调试Python进程
游戏进程一般是 C++ 引擎调用 Python 解释器的方式运行，因此实际的进程是 C++的。
但是如果我们有个 Python 进程，那么就可以使用`gdb`+`python-dbg`的方式进行调试。

首先安装`python-dbg`：
```shell
sudo apt-get install gdb python3.6-dbg
```
然后下载`libpython`包到文件系统中，最后采用`gdb`链接正在运行的进程，然后按以下步骤调试：
```shell
(gdb) python
>import sys
>sys.path.insert(0, '/home/xxx')
>import libpython
>end


(gdb) py-bt

Traceback (most recent call first):
File "run_forever_block.py", line 10, in simple_server
data, addr = s.recvfrom(2048)
File "run_forever_block.py", line 17, in <module>
　　　　simple_server()
```
`libpython`中常见的命令有：
```shell
py-list 　　查看当前python应用程序上下文
py-bt 　　 查看当前python应用程序调用堆栈
py-bt-full 　　 查看当前python应用程序调用堆栈，并且显示每个frame的详细情况
py-print 　　 查看python变量
py-locals　　 查看当前的scope的变量
py-up 　　 查看上一个frame
py-down 　　 查看下一个frame
```
## 进程陷入死循环
```shell
rxsi@VM-20-9-debian:~$ top
top - 16:19:48 up 138 days,  5:22,  0 users,  load average: 0.64, 0.17, 0.06
Tasks: 112 total,   2 running, 110 sleeping,   0 stopped,   0 zombie
%Cpu(s): 25.0 us,  0.2 sy,  0.0 ni, 74.5 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem :   3818.4 total,    554.2 free,    619.1 used,   2645.1 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2886.8 avail Mem 

PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                                          
13578 rxsi      20   0    6564   1728   1568 R  99.7   0.0   0:51.99 test_circle_block    
```
如果进程进入了死循环状态，那么就会处于`R`状态，并且占用了近乎所有`CPU`资源（当然如果程序本身是 CPU 密集型程序也是可能的。。。）
此时使用`strace`是无法查看进程死循环原因的，只能使用`gdb`命令
```shell
rxsi@VM-20-9-debian:~$ gdb program -p 13578
```
然后依然调用`bt`显示堆栈信息
```shell
(gdb) bt
#0  0x000055cecc2a1129 in fun1() ()
#1  0x000055cecc2a1140 in main ()
```
这时候就需要分析为何`fun1()`函数是否存在死循环问题
## 多进程/多线程死锁
如果进程发生死锁，那么进程会处于`S`状态，因此依然可以使用`strace`或者`gdb`命令进行分析

运行`thread_test`可执行文件，这个文件会运行两个进程并形成死锁
```shell
rxsi@VM-20-9-debian:~/learncpp$ ./thread_test &
[1] 14829
rxsi@VM-20-9-debian:~/learncpp$ 
0x7ffe2db36700: lock in mutext
0x7ffe2db366f8: lock in mutext2
0x7ffe2db36700: thread running
0x7ffe2db366f8: thread running
0x7ffe2db36700: try to lock mymutext2
0x7ffe2db366f8: try to lock mymutext

rxsi@VM-20-9-debian:~/learncpp$ ps -aux | grep thread_test
rxsi     14829  0.0  0.0  88624  1792 pts/0    Sl   19:48   0:00 ./thread_test
rxsi     15519  0.0  0.0   7256   912 pts/0    S+   20:27   0:00 grep thread_test
```
使用`gdb`命令链接到进程，并打印当前的堆栈信息
```shell
rxsi@VM-20-9-debian:~/learncpp$ gdb program -p 14829
//..........
(gdb) info thread 
Id   Target Id                                       Frame 
* 1    Thread 0x7fd65a842740 (LWP 14829) "thread_test" 0x00007fd65ad36495 in __pthread_timedjoin_ex () from /lib/x86_64-linux-gnu/libpthread.so.0
2    Thread 0x7fd65a841700 (LWP 14830) "thread_test" 0x00007fd65ad3e29c in __lll_lock_wait () from /lib/x86_64-linux-gnu/libpthread.so.0
3    Thread 0x7fd65a040700 (LWP 14831) "thread_test" 0x00007fd65ad3e29c in __lll_lock_wait () from /lib/x86_64-linux-gnu/libpthread.so.0
(gdb) thread 2
[Switching to thread 2 (Thread 0x7fd65a841700 (LWP 14830))]
#0  0x00007fd65ad3e29c in __lll_lock_wait () from /lib/x86_64-linux-gnu/libpthread.so.0
(gdb) bt  
#0  0x00007fd65ad3e29c in __lll_lock_wait () from /lib/x86_64-linux-gnu/libpthread.so.0
#1  0x00007fd65ad37714 in pthread_mutex_lock () from /lib/x86_64-linux-gnu/libpthread.so.0
#2  0x000055d30aac92c0 in thread_fun(void*) ()
#3  0x00007fd65ad34fa3 in start_thread () from /lib/x86_64-linux-gnu/libpthread.so.0
#4  0x00007fd65a944eff in clone () from /lib/x86_64-linux-gnu/libc.so.6
(gdb) 
```
从上面的输出可以知道，此时子进程正阻塞在`__lll_lock_wait`函数，等待着锁的释放
## 让进程生成coredump文件
在《线程与进程》中有介绍了 linux 系统信号类型，其中有一些信号的产生会使进程终止并生成`coredump`文件，因此对于严重阻塞生产环境的进程，可以使之先生成`coredump`文件，然后再杀掉进程，再重启进程，而进程异常的原因就可以从`coredump`文件中分析得知。

默认情况下 linux 系统是不允许生成`coredump`文件的，限制了文件的大小为 0，从下面的指令可知：
```shell
rxsi@VM-20-9-debian:~$ ulimit -c
0 
```
因此需要先调整`coredump`文件的大小限制，这里可以先设置为`unlimited`
```shell
rxsi@VM-20-9-debian:~$ ulimit -c unlimited
rxsi@VM-20-9-debian:~$ ulimit -c
unlimited
```
不过这个命令只能存在于当前的会话，因此我们可以临时建立个会话调整并生成`coredump`文件之后就退出会话，这样就可以使配置回复原样了

如果要永久修改配置，则需要通过下面的方式
```shell
rxsi@VM-20-9-debian:~$ sudo vim /etc/security/limits.conf

```
将文件中这个注释去掉，并修改`coredump`文件的大小

![coredump.png](/images/linux_process_kill/coredump.png)

生成的`coredump`文件默认存在于当前执行命令的位置
```shell
rxsi@VM-20-9-debian:~$ cat /proc/sys/kernel/core_pattern 
core
```
也可以通过下面命令自定义路径
```shell
echo "core-%e-%p-%s-%t" > /proc/sys/kernel/core_pattern

%p  出Core进程的PID
%u  出Core进程的UID
%s  造成Core的signal号
%t  出Core的时间，从1970-01-0100:00:00开始的秒数
%e  出Core进程对应的可执行文件名
```
### kill命令
使用`kill`命令向进程发出会使其终止并生成`coredump`文件的信号，`11`是`SIGSEGV`信号
```shell
rxsi@VM-20-9-debian:~$ kill -11 19694
```
### gdb命令
先使用`gdb`连接入进程，再使之生成`coredump`文件，然后再`kill`掉进程
```shell
rxsi@VM-20-9-debian:~$ gdb program -p 20278

(gdb) gcore
Saved corefile core.20278

// 退出gdb之后
rxsi@VM-20-9-debian:~$ kill -9 20278
```
### 分析coredump文件
要分析`coredump`文件，首先需要对源文件加上`-g`参数后编译，然后再使用下面的命令：
```shell
rxsi@VM-20-9-debian:~$ gdb ./learncpp/test_circle_blocking core.20278 
GNU gdb (GDB) 8.3.1
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-pc-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./learncpp/test_circle_blocking...

warning: core file may not match specified executable file.
[New LWP 20278]
Core was generated by `./test_circle_blocking'.
#0  fun1 () at test_circle_blocking.cpp:4
4               while(true){}   
```
可以看出`coredump`时正在调用的函数，然后就可以配合`gdb`的其他命令进行分析了。
## 查看进程被kill原因
当进程在运行过程中被系统`kill`时，可以通过`dmesg`指令查看原因

运行`helloworld`可执行程序，一开始进程宕机的原因是`1/0`，后面再修改为内存溢出，指令输出如下：
```shell
rxsi@VM-20-9-debian:~$ sudo dmesg -T
# .....
[Thu Sep 29 20:44:29 2022] traps: helloworld[23862] trap divide error ip:56226e4d51b5 sp:7ffc588645a0 error:0 in helloworld[56226e4d5000+1000]
[Fri Sep 30 10:17:19 2022] helloworld[3414]: segfault at 0 ip 00007fa569c0a5dc sp 00007fff5e094cd8 error 6 in libc-2.28.so[7fa569ad0000+147000]
[Fri Sep 30 10:17:19 2022] Code: 9d 48 81 fa 80 00 00 00 77 19 c5 fe 7f 07 c5 fe 7f 47 20 c5 fe 7f 44 17 e0 c5 fe 7f 44 17 c0 c5 f8 77 c3 48 8d 8f 80 00 00 00 <c5> fe 7f 07 48 83 e1 80 c5 fe 7f 44 17 e0 c5 fe 7f 47 20 c5 fe 7f
```
