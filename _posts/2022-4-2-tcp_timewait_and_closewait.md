---
layout: post
title: TIME_WAIT&CLOSE_WAIT
date: 2022-4-2 15:36:57 +0800
categories: TCP
tags: tcp TIME_WAIT CLOSE_WAIT
author: Rxsi
---

* content
{:toc}


## TIME_WAIT 状态等待 2MSL 的原因
客户端发出 ACK 报文时，最大化时间考虑的假设是 **“该ACK报文经过了一个最大化的报文生存时间（MSL）才被服务端接收”**，如果没有被接收到，那么服务端会重新下发 FIN 报文，那么假设也是 **“经过了最大化的报文生成时间（MSL）才到达客户端”**，那么这样一来一去，也就是需要 2MSL 了。在 linux 系统中，这个时间默认是`60s`。
**注意：当客户端重新接收到 FIN 报文时，2MSL 会重新计时**

## TCP TIME_WAIT状态过多
### TIME_WAIT 状态的作用：

- 1 可靠的实现TCP全双工连接的终止
- 2 确保重用相同IP、端口的连接后，旧的数据包已经消亡

### TIME_WAIT 状态过多的危害：

- 1 占用内存资源，虽然在 TCP 转为`TIME_WAIT`状态之后会用一个轻量的数据结构保存地址端口等必要信息，但是如果存在大量的这些数据还是对内存有一定的影响
- 2 占用端口资源，当端口资源被占用满了，会导致无法创建新的连接

## TIME_WAIT 状态过多的产生原因：

- 1 高并发时产生过多的短连接
- 2 默认的2MSL时间太长

## TIME_WAIT 状态的优化：

- 1 使用长连接，但是会增加资源占用

- 2 增加服务端对外服务端口，增加客户端机网卡，但是治标不治本

- 3 调整内核参数：

    1）**tcp_tw_reuse 和 tcp_timestamp** 
    ```c
    tcp_tw_reuse = 1 // 是否开启重用正处于TIME_WAIT状态的socket，默认为0，表示否
    tcp_timestamp = 1 // 开启TCP时间戳，默认开启
    ```
    只能针对**客户端**有效，当开启该功能时，在调用 connect() 函数时，内核会随机找一个TIME_WAIT状态 **超过1秒** 的socket给新的连接复用。（谨记：服务端也是可以主动断开连接而进入TIME_WAIT状态的，当然一般设置不主动关闭连接，除非是HTTP请求。）

	2）**tcp_max_tw_buckets** 
    ```c
    tcp_max_tw_buckets = 18000 // 最大的TIME_WAIT数量
    ```
    将最大的TIME_WAIT数量值调小，当检测到TIME_WAIT数量超过限定值之后，所有的TIME_WAIT状态都会被立即清除。但是做法粗暴，有造成异常的风险

    3）修改等待2MSL的时间，需要重新编译内核

    4）**SO_REUSEADDR** 参数
    ```c
    int on = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (char*)&on, sizeof(on));
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEPORT, (char*)&on, sizeof(on));
    ```
    开启该参数之后，即可重用处于 **TIME_WAIT** 状态的 socket，一般用在服务端（注意 **SO_REUSEPORT**是为了解决特定IP端口的复用问题，但是却无法绑定处于 TIME_WAIT 状态的 socket，因此一般同时使用这两个参数）。通过该参数才可以实现服务的快速重启，否则重启之后会无法对端口进行监听

## TCP CLOSE_WAIT状态过多
### CLOSE_WAIT状态的危害：
占用fd套接字资源（可使用selelct_server.cpp测试，nc模拟客户端），如果是进入了LAST_ACK状态，则不会占用资源

### CLOSE_WAIT产生的原因：
一般这种情况属于程序中未正确的调用close()函数

### CLOSE_WAIT优化：
检查程序运行代码，特别是一些对端异常断开时，是否有在对应的try..catch中进行资源的释放