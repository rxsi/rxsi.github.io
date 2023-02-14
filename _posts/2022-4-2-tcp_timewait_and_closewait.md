---
layout: post
title: TIME_WAIT&CLOSE_WAIT
date: 2022-4-2 15:36:57 +0800
categories: TCP
tags: TIME_WAIT CLOSE_WAIT
author: Rxsi
---

* content
{:toc}

## TIME_WAIT
当 TCP 主动发起挥手并成功关闭之后，则会进入`TIME_WAIT`状态
### TIME_WAIT状态的作用
之所以设计一个长时间（2MSL，linux 中默认是 60s）的等待状态，主要有两个目的：
1. 确保连接的可靠正常关闭

    在第四次挥手阶段发出的`ACK`报文是可能会丢失的，因此需要有一个缓冲期用以等待对端重发`FIN`报文，否则如果本端直接进入了`CLOSED`状态，那么对于对端重发的`FIN`报文会直接回复`RST`报文，那么对于对端来说即使所有数据都正确发送了，仍然会是以一种“异常”的方式结束
    
2. 确保重用相同 IP、端口的连接不会接收到上个连接的旧数据包

    如果在重用之后网络环境中还存在上条连接的旧数据包，那么可能会错认为这些数据包是当前连接的，而造成错误的处理。但是实际上这种情况的几率极小，因为至少要满足两点，一是使用了相同的IP、端口信息（客户端的端口一般是随机的），二是该数据包的序列号正好在新连接中是有效的（服务端的序列号、客户端的序列号都是重新随机的）。所以`TIME_WAIT`状态预防的就是这种极端情况下可能出现的异常问题，状态的持续时间是`2MSL`，在 linux 系统中则是为`60s`。
    
    如果开启了`SO_REUSEADDR`和`SO_REUSEPORT`那么新建立的连接就可以立即结束上一个就连接的`TIME_WAIT`状态进行IP、端口的重用，这会使得接收到上条连接的旧数据包的概率增大，如果一条新连接接收到了旧数据包，那么极大概率会因为序列号匹配失败而回复正确的`ACK`报文，而对端接收到了正确的`ACK`报文不会造成任何异常，因此连接可以正常进行。

**注意：当客户端重新接收到 FIN 报文时，2MSL 会重新计时**

<!--more-->
### TIME_WAIT状态过多的危害：
1. 占用内存资源，虽然在 TCP 转为`TIME_WAIT`状态之后会用一个轻量的数据结构保存地址端口等必要信息，但是依然占有文件描述符，而一个进程所能拥有的文件描述符是由限制的；
2. 占用端口资源，当端口资源被占用满了，会导致无法创建新的连接，一台机器的可用端口是有限的，在 linux 系统的可用端口配置如下：
    ```shell
    rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/ip_local_port_range 
    32768   60999
    ```

### TIME_WAIT状态过多的产生原因：
1. 高并发时产生过多的短连接
2. 默认的 2MSL 时间太长

### TIME_WAIT状态的优化：
1. 使用长连接，但是会增加资源占用
2. 增加服务端对外服务端口，增加客户端机网卡，但是治标不治本
3. 调整内核参数：
    - tcp_tw_reuse 和 tcp_timestamp： 
    ```c
    tcp_tw_reuse = 1 // 是否开启重用正处于TIME_WAIT状态的socket，默认为0，表示否
    tcp_timestamp = 1 // 开启TCP时间戳，默认开启
    ```
    只能针对**客户端**有效，当开启该功能时，在调用`connect()`函数时，内核会随机找一个 TIME_WAIT 状态**超过1秒** 的socket给新的连接复用。（谨记：服务端也是可以主动断开连接而进入TIME_WAIT状态的，当然一般设置不主动关闭连接，除非是HTTP请求。）

	- tcp_max_tw_buckets 
    ```c
    tcp_max_tw_buckets = 18000 // 最大的TIME_WAIT数量
    ```
    将最大的TIME_WAIT数量值调小，当检测到TIME_WAIT数量超过限定值之后，所有的TIME_WAIT状态都会被立即清除。但是做法粗暴，有造成异常的风险

    - 修改等待 2MSL 的时间，需要重新编译内核

    - SO_REUSEADDR：
    ```c
    int on = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (char*)&on, sizeof(on));
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEPORT, (char*)&on, sizeof(on));
    ```
    开启该参数之后，即可重用处于`TIME_WAIT`状态的 socket，一般用在服务端（注意 SO_REUSEPORT 是为了使多个 socket 同时绑定相同的 IP、端口，但是却无法绑定处于`TIME_WAIT`状态的 socket，因此一般同时使用这两个参数）。通过该参数才可以实现服务的快速重启，否则重启之后会无法对端口进行监听

    ` SO_LINGER：
    这个参数可以使调用 close 时直接下发`RST`报文，而粗暴的直接关闭 tcp，并不推荐使用。

## CLOSE_WAIT
当系统接收到了第一次挥手包并回复第二次挥手包后就会进入`CLOSE_WAIT`状态
### CLOSE_WAIT状态的作用
这个状态是用来让被动关闭方能够正确的处理完接收到的数据并发送完所要发送的数据
### CLOSE_WAIT状态过多的危害
处于此状态的 socket 会占用进程的套接字资源，而一个进程能够拥有的套接字资源是有上限的，由以下配置控制：
```shell
rxsi@VM-20-9-debian:~$ ulimit -n
1024
```
如果进入了`LAST_ACK`状态，那么就不会占用套接字资源了。（可使用selelct_server.cpp测试，nc模拟客户端）

### CLOSE_WAIT状态过多的原因
一般这种情况属于程序中未正确的调用`close()`函数

### CLOSE_WAIT状态过多的优化
检查程序运行代码，特别是一些对端异常断开时，是否有在对应的 try..catch 中进行资源的释放