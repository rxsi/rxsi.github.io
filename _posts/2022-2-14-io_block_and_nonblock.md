---
layout: post
title: 阻塞IO和非阻塞IO
date: 2022-2-14 13:16:43 +0800
categories: IO技术
tags: io 阻塞 非阻塞 
author: Rxsi
---

* content
{:toc}

### file descriptor（fd）
在 linux 系统中，使用 **task_struct** 管理一个进程，其中就有 **files_struct** 结构，该结构负责管理进程所拥有的文件描述符。

#### files_struct
```c
struct files_struct { 
	...
	struct fdtable  *fdt;  
	struct fdtable  fdtab;  
	...
};
```
<!--more-->
每个 **files_struct** 都含有 **fdtable** 结构，本质上是文件描述符的集合，因此返回的文件描述符是一个int类型，表示的就是这个集合的下标。
默认情况下：

- 0：stdin，标准输入
- 1：stdout，标准输出
- 2：stderr，标准错误

#### 重定向
底层使用的是 dup() 和 dup2()方法，本质上是把某个文件描述符值赋值给 **fdtable** 其他下标对应的文件描述符
常见形式有：

-  [n] < file：n不填时默认为0，即标准输入 
-  [n] > file：n不填时默认为1，即标准输出  

```shell
echo "hello" > file1;
./test < file1;
```

### 磁盘IO的特殊性
磁盘IO不同于网络IO，磁盘IO只能是阻塞的，也就是并不能将其设置为`O_NONBLOCK`，对其`read/write`也不会返回`EAGAIN`。这是POSIX系统对磁盘文件（regular files）的底层设计，当监听对应的磁盘`fd`时，总是返回`Ready`状态，但实际读取时如果文件数据不在内存缓存中，则read操作本身还是会“**阻塞**”等待数据从磁盘读出。
**由于这个原因，`EPOLL`不支持监听磁盘IO。**
```python
f = open('node.py')
fd = f.fileno()
while True:
    r, w, e = select.select([fd], [], []) # 使用select可以运行
    print '>', repr(os.read(fd, 10))
    time.sleep(1)
    
self._impl.register(fd, events | self.ERROR) # 使用epoll将会报错
# IOError: [Errno 1] Operation not permitted 
```

### Socket API
以BSD Socket为标准（伯克利套接字）

#### 客户端API
##### int socket(int af, int type, int protocol);

- 参数1：af 为地址族，也就是IP地址的类型，有 AF_INET 和 AF_INET6
- 参数2：type 为套接字类型，常用的有 SOCK_STREAM 和 SOCK_DGRAM
- 参数3：protocol 为传输协议，常用的有 IPPROTO_TCP 和 IPPROTO_UDP，还有 0。当由前两种组合可以确定只有一种协议满足时，可以直接填0，比方说 int clientfd = socket(AF_INET, SOCK_STREAM, 0);
- 返回值：返回的是socket类型，在linux平台上，socket是int类型，而在windows平台上是 SOCKET 宏类型，本质上也是一个int。当返回值为-1时，表示申请socket套接字失败，否则为大于0的值。具体的错误信息可以由errno中获取。

```c
int clientfd = socket(AF_INET, SOCK_STREAM, 0);
if (clientfd == -1){
    std::cout << "create client socket error. " << std::endl;
    return -1;
}
```

##### int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

-  参数1：系统调用 socket() 函数返回的套接字 
-  参数2：保存着目标服务器IP端口信息的结构体。一般是通过sockaddr_in（in表示的是ipv4， 常见的还有sockaddr_in6，sockaddr_un等）指针转换为sockaddr指针 
-  参数3：addr变量的大小，可由sizeof计算。在这里该参数不是一个指针，但是在accept()函数中则需要的是一个指针类型。 
-  返回值：当调用失败时返回-1，且错误信息可由errno中获取 
-  在阻塞和非阻塞IO下调用方式有区别：
   - 阻塞：connect对应的是TCP三次握手的发送SYN操作（CLOSE -> SYS_SEND），因此在阻塞模式下会等待服务端返回的SYN-ACK报文（SYS_SEND -> ESTABLISH），至少阻塞一个RTT时间。
    ```cpp
    // 阻塞模式下
    sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS); // 解析服务端ip地址
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (sockaddr*)&serveraddr, sizeof(serveraddr)) == -1){
        std::cout << "connect socket error." << std::endl;
        close(clientfd);
        return -1;
    }
    ```

   - 非阻塞： 非阻塞模式下，调用会立即返回，需要在实际利用对应clientfd之间，借助select或者poll检测是否已经可写，并且在linux平台还需要额外检测判断该socket是否报错（在socket报错的情况下，select和poll也会返回可写状态，即返回1）  
    ```cpp
    // 非阻塞模式下
    sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS); // 解析服务端ip地址
    serveraddr.sin_port = htons(SERVER_PORT);
    while (true){
        int ret = connect(clientfd, (sockaddr*)&serveraddr, sizeof(serveraddr));
        if (ret == 0){
            std::cout << "connect to server successfully." << std::endl;
            break;
        } else if (ret == -1){
            if (errno == EINTR){ // 这里是因为系统没有资源，所以暂时不能执行，因此我们需要使用循环继续判断
                std::cout << "connecting interruptted by signal, try again." << std::endl;
                continue;
            } else if (errno == EINPROGRESS){ // 这里是因为已经开始建立连接了，因为是non-blocking的原因，因此立即返回了，接下来只需要在select/poll中判断是否是可写事件，最后还需要使用getsockopt判断连接是否成功
                std::cout << "auto try connect now, don't need to try again." << std::endl;
                break;
            } else{
                close(clientfd);
                std::cout << "connect error." << std::endl;
                return -1;
            }
        }
    }
    fd_set writeset;
    FD_ZERO(&writeset);
    FD_SET(clientfd, &writeset);
    timeval tv;
    tv.tv_sec = 3;
    tv.tv_usec = 0;
    if (select(clientfd+1, NULL, &writeset, NULL, &tv) != 1){ 
        std::cout << "[select] connect to server error." << std::endl;
        close(clientfd);
        return -1;
    }
    // 补充判断阶段:
    int err;
    socklen_t len = static_cast<socklen_t>(sizeof(err));
    if(::getsockopt(clientfd, SOL_SOCKET, SO_ERROR, &err, &len) < 0){
        close(clientfd);
        return -1;
    }
    if (err == 0){
        std::cout << "connect to server successfully." << std::endl;
    } else{
        std::cout << "connect to server error." << std::endl;
        close(clientfd);
        return -1;
    }
    ```

##### ssize_t send(int sockfd, const void *buf, size_t len, int flags);
相比于`ssize_t write(int fd, const void* buf, size_t nbytes)`多了一个标志位，但是`write`可用于向管道写入数据

- 参数1：套接字
- 参数2：待发送的数据缓存
- 参数3：指明buf的长度
- 参数4：标志位，一般填0
- 返回值：当处于阻塞模式下时，会阻塞到可以完整发送数据时才返回，如当对端的接收窗口过小时，会导致一直阻塞，如果返回的值不等于len，则代表发送失败；在非阻塞模式下，需要考虑成功发送一部分（0 < ret < len），-1 和 ==len 的情况。**当=0时，一般意味着连接断开，如果是主动发送0字节长度的数据，虽然返回值依然是0但是底层会过滤掉，不会发送给对端，因此我们一般使用是否等于len去判断，而不是直接判断0**
- 在阻塞和非阻塞IO下调用方式有区别：
   - 阻塞：在阻塞模式下，会一直阻塞到发送完预设的数据长度，除非发生异常
    **本质上是两次阻塞：1.阻塞等待有足够的写入空间；2.阻塞进行数据的从用户态到内核态的copy**。
    ```c
    int ret = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0); // 这里使用的是阻塞模式,当无法发送时,会阻塞在这里.
    if (ret != strlen(SEND_DATA)){
        std::cout << "send data error." << std::endl;
        break;
    } else{
        std::cout << "send data successfully, count = " << count << std::endl;
    }
    ```
 

   - 非阻塞：在非阻塞模式下，要考虑发送是否被中断（INTER），或者因窗口过小而暂时无法发送（EWOULDBLOCK、EAGAIN），且需要使用循环判断是否完整的发送完数据
    **非阻塞模式下，只有copy时的一次阻塞** 
    ```c
    while (true){
        if (SendData(clientfd, SEND_DATA, strlen(SEND_DATA, 0))){
            std::cout << "send data successfully " << std::endl;
        } else {
            std::cout << "send data error." << std::endl;
            break;
        }
    }
    close(clientfd);
    // 发送函数
    bool SendData(int fd, const char* buf, int buf_len, int flag = 0){
        int send_len = 0;
        int ret = 0;
        while (true){
            ret = send(fd, buf + send_len, buf_len - send_len, flag);
            if (ret == -1){
                if (errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN){
                    std::cout << "TCP Window size is too small or interrupted by single" << std::endl;
                    continue;
                } else{
                    std::cout << "send data error." << std::endl;
                    return false;
                }
            } else if (ret == 0){
                std::cout << "send data error." << std::endl;
                return false;
            }
            send_len += ret;
            if (send_len == buf_len){
                return true;
            } else {
                return false; 
            }
        }
    }
    ```

##### ssize_t recv(int sockfd, void *buf, size_t len, int flags);
相比于`ssize_t read(int fd, void* buf, size_t nbyte)`多了一个标志位，但是`read`函数可用于从管道套接字读取数据

- 参数1：套接字
- 参数2：接收缓冲区
- 参数3：接收缓冲区的大小
- 参数4：标志位，一般为0
- 返回值：在阻塞模式下，=0意味着连接断开，-1异常，>0接收到数据（注意不一定等于预设的len）；非阻塞模式下，-1时需要考虑中断情况
- 在阻塞和非阻塞IO下调用方式有区别：
   - 阻塞：
    **本质上是两次阻塞：1.阻塞等待有可读数据；2.阻塞进行数据的从内核态到用户态的copy** 
    ```c
    char buf[32] = {0}; 
    int ret = recv(clientfd, buf, 32, 0); // 当没有接收到数据时,会阻塞在这里
    if (ret > 0){
        std::cout << "recv successfully." << std::endl;
    } else{
        std::cout << "recv data error." << std::endl;
    }
    ```

   - 非阻塞：
    **只有copy时的一次阻塞** 
    ```c
    while (true){
        char recvbuf[32] = {0}; 
        int ret = recv(clientfd, recvbuf, 32, 0); 
        if (ret > 0){
            std::cout << "recv successfully." << std::endl;
        } else if (ret == 0){ 
            std::cout << "peer close the socket." << std::endl;
            break;
        } else if (ret == -1){
            if (errno == EINTR || errno == EAGAIN || errno == EWOULDBLOCK){
                std::cout << "There is no data avaliable now or interrupted" << std::endl;
            } else{
                break;
            }
        }
    }
    close(clientfd);
    ```

#### 服务端API
##### int socket(int af, int type, int protocol);
与客户端同

##### ssize_t send(int sockfd, const void *buf, size_t len, int flags);
与客户端同

##### ssize_t recv(int sockfd, void *buf, size_t len, int flags);
与客户端同

##### int bind(int sock, struct sockaddr *addr, socklen_t addrlen);

- 参数1：套接字
- 参数2：保存着目标服务器IP端口信息的结构体。一般是通过sockaddr_in指针转换为sockaddr指针
- 参数3：addr的大小
- 返回值：-1表示错误

```c
// 初始化服务器地址
sockaddr_in bindaddr;  // C++中struct可以不加
bindaddr.sin_family = AF_INET;

// htonl (h: host; to: to; n: net; l: unsigned long) 意思是将计算机的内存顺序转换成网络字节顺序(也就是大端序\小端序调整)
// htons 也是一样的作用

// INADDR_ANY表示应用程序不关心bind绑定的ip,由底层自动选择一个合适的ip地址,适合在多网卡机器上选择ip,相当于0.0.0.0
// 如果只是想在本机上访问,则绑定ip可以使用127.0.0.1
// 局域网中的内部机器访问,则绑定机器的局域网ip
// 公网访问则需要是0.0.0.0或者 INADDR_ANY
bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
// 服务端只需要bind一个固定的端口,而如果是客户端则不能,否则同台机器将会无法启动多个客户端,因此对于客户端来说不进行bind或者bind(0),由底层自动分配
// 原因:tcp使用4元组进行区分,server虽然之后一个port,但是连接的client有不同的ip和port,因此可以构成不同的四元组.
bindaddr.sin_port = htons(3000);

if (bind(listenfd, (sockaddr*)&bindaddr, sizeof(bindaddr)) == -1){
    std::cout << "bind listen socket error." << std::endl;
    close(listenfd);
    return -1;
}
```

##### int listen(int sock, int backlog);

- 参数1：套接字
- 参数2：backlong参数，一般使用SOMAXCONN参数
作用1：决定全连接队列的上限 = min(SOMAXCONN, backlog)；
作用2：决定半连接队列的上限 = min(SOMAXCONN, backlog, tcp_max_syn_backlog)
- 返回值：-1表示失败

```c
// 如果设置为SOMAXCONN则是由系统决定(可能是几百),当请求队列满了客户端会接收到ECONNREFUSED错误(linux)/ WSAECONNREFUSED错误(windows) 
if (listen(listenfd, SOMAXCONN) == -1){
    std::cout << "listen error." << std::endl;
    close(listenfd);
    return -1;
}
```

##### int accept(int sock, struct sockaddr addr, socklen_t addrlen);

-  参数1：套接字 
-  参数2：保存着目标服务器IP端口信息的结构体。一般是通过sockaddr_in指针转换为sockaddr指针。**注意由于上个socket函数一般是bind()函数，也有sockaddr指针，但是这里要使用一个空的sockaddr结构体指针，而不是复用bind()函数的那个。** 
-  参数3：指向socklen_t的指针，本质就是一个int指针 
-  返回值：-1表示错误 
-  在阻塞和非阻塞IO下调用方式有区别：
    - 阻塞： 
    ```c
    sockaddr_in clientaddr;
    socklen_t clientaddrlen = sizeof(clientaddr);
    int clientfd = accept(listenfd, (sockaddr*)&clientaddr, &clientaddrlen);
    if (clientfd != -1){ // 如果后续不调用recv函数，那么服务端的接收窗口会一步步被塞满，而最终导致客户端的发送窗口也被塞满。
    std::cout << "accept a client connection. " << std::endl;
    }
    ```

    - 非阻塞：  
    ```c
    while (true){
        sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        int clientfd = accept(listenfd, (sockaddr*)&clientaddr, &clientaddrlen);
        if (clientfd == -1){
            if (errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN){
                std::cout << "empty accept queue or interrupted by single" << std::endl;
                continue;
            } else{
                std::cout << "accept error." << std::endl;
                break; // 这里直接退出了，可能会导致会面的accept接收队列不被处理，需要谨慎。
            }
        } else {
            std::cout << "accept successfully." << std::endl;
            // 将该clientfd保存起来，以作后续操作
        }
    }
    ```

#### readv 和 writev 函数

- ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
- ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
- ssize_t preadv(int fd, const struct iovec *iov, int iovcnt, off_t offset);
- ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt, off_t offset);

用以将多个缓冲区数据同时写入一个fd套接字，使用`read/write`则需要循环调用，造成额外的性能开销。
```c
struct iovec {
    void *iov_base; // 数据起始地址
    size_t iov_len; // 数据要传输的字节数
}
```

实例代码：
```c
char *info1 = "time to sleep";
char *info2 = "go to work";

struct iovec iov[2];
iov[0].iov_base = info1;
iov[0].iov_len = strlen(info1);

iov[1].iov_base = info2;
iov[1].iov_len = strlen(info2);

ssize_t nwritten = writev(fd, iov, 2); // 返回成功写入的字节数
```

#### 总结

- 阻塞模式和非阻塞模式下，有影响的API有：**connect、accpet、send、recv**
- 阻塞模式和非阻塞模式的最大区别是，当非阻塞模式下读取或写入数据失效时，会返回响应的错误码，这在IO多路复用上有极大的应用。比方说，在epoll的LT模式下，某个fd有可读数据时会一直触发EPOLLIN事件信号。假设数据超过了一次读取预设的接收缓冲区，当使用阻塞的recv，我们需要等待每次的EPOLLIN事件才进行一次读取，如果强制使用循环去读则有可能在某次读取把剩余数据读取完之后，再读取时发生recv空数据，造成阻塞，显而易见这种做法效率不高。如果使用的是非阻塞的recv，就可以在一次EPOLLIN事件里面，循环读取完，直到触发EAGAIN或EWOULDBLOCK信号再退出，效率更高。
而且，根据 man select 中的描述，有可能出现提示有可读事件，但是由于校验和不通过等原因而丢弃数据的情况，因此需要使用非阻塞IO
### 主机字节序和网络字节序
#### 主机字节序
以 0x10203040 存储到 4001 开始的内存位置为例：

- 大端序：数据的低位字节放在内存的高位，高位字节放在内存的低位，比较符合人类的阅读习惯
```c
4001 4002 4003 4004
10    20    30    40
```

- 小端序：数据的低位字节放在内存的低位，高位字节放在内存的高位
```c
4001 4002 4003 4004
40    30    20    10
```

#### 网络字节序（TCP/IP规定的字节序）
规定为 **大端序**，因此当传入 TCP 时，需要进行字节序转换
```
uint32_t htonl(uint32_t netlong); // 将long类型转换为网络字节序
uint16_t htons(uint16_t netshort); // 将short类型转换为网络字节序
```

**判断本机字节序的方式：**
使用一个2字节的十六进制数，如 unsigned short num = 0x1234，然后将其强行转换为1字节的char类型，会把高字节部分丢弃，如 char_ mode = (char_)&num。
也就是如果是小端序，那么结果是34，大端序则为12
### SO_REUSEADDR 和 SO_REUSEPORT
#### SO_REUSEADDR
默认情况下，一个 Socket 以五元组作为唯一标识：`{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}`。
对于源IP来说，`0.0.0.0`代表着绑定本机所有网卡地址，对应到C++代码中则为：（一般用在服务端）
```c
sockaddr_in bindaddr;
bindaddr.sin_family = AF_INET;
bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
```
对于源port来说，`0`代表的是绑定随机的某个端口，对应到C++代码中则为：（一般用在客户端）
```c
int clientfd = socket(AF_INET, SOCK_STREAM, 0);
```
因此在没有 **SO_RESUEADDR** 配置下，`0.0.0.0:21` 和 `192.168.0.1:21` 将会绑定失败。

| SO_REUSEADDR | socketA | socketB | Result |
| --- | --- | --- | --- |
| ON/OFF | 192.168.0.1:21 | 192.168.0.1:21 | Error |
| ON/OFF | 192.168.0.1:21 | 10.0.0.1:21 | OK |
| ON/OFF | 10.0.0.1:21 | 192.168.0.1:21 | OK |
| OFF | 0.0.0.0:21 | 192.168.0.1:21 | Error |
| OFF | 192.168.0.1:21 | 0.0.0.0:21 | Error |
| ON | 0.0.0.0:21 | 192.168.0.1:21 | OK |
| ON | 192.168.0.1:21 | 0.0.0.0:21 | OK |
| ON/OFF | 0.0.0.0:21 | 0.0.0.0:21 | Error |


同时当 TCP 主动关闭时，会进行 **TIME_WAIT** 状态，正常情况下会使程序无法在一段时间（2MSL）内无法重用对应的IP和端口。开启 **SO_REUSEADDR** 后即可进行重用。
在 BSD 上使用 **SO_REUSEADDR** 参数不需要关心其他 socket 是否设置了相同的 **SO_REUSEADDR** 参数，只要一方有设置，则设置方便可生效。**但是在 linux 上必须要所有 socket 都设置**。

#### SO_REUSEPORT
在`socket`服务端开发的过程中，我们遵循的步骤一般为：

1. 调用`socket()`函数创建一个`listenfd`
2. 为该`listenfd`调用`bind()`函数绑定一个`IP + PORT`
3. 使用`listen()`函数监听该`listenfd`
4. 调用`accept()`函数阻塞/非阻塞的创建一个`clientfd`

在早期的开发流程中，一般有两种多进程的开发模式：

1. 使用的是一个父进程去调用`socket() + bind() + listen() + accept()`，当成功`accept`了一个`clientfd`之后，再`fork`出一个子进程去处理这个`clientfd`。
2. 使用的是一个父进程去调用`socket() + bind() + listen()`，然后`fork`出多个子进程去同时调用`accept()`函数对这个`listenfd`进行`accept`操作，这种做法早期会有惊群问题，但是在`linux2.6`之后，当多个进程同时调用`accept`同个`listenfd`时，使用的是`prepare_to_wait_exclusive`的**互斥**方式将所有的进程添加到`listenfd`的监听队列中，因此不再有惊群现象。

在上面的两种形式，终究都是为每一个`clientfd`创建一个处理进程，而 IO 事件实际上往往只是一瞬间的事情，因此在处理完成之后，要么阻塞的等待下一次事件的到来，要么就销毁进程/线程，可见这种做法有严重的性能问题
为了使用提升性能，避免频繁的开创和销毁进程/线程，我们通常在一个进程中使用 IO 多路复用 + 非阻塞套接字的形式，以提升性能。一般来说我们会使用`EPOLL`去操作这些套接字，此时的做法演变为一个进程调用`socket() + bind() + listen() + epoll_create()`，然后使用`epoll_wait`去等待这个`listenfd`的事件。当这个`listenfd`有`EOPLLIN`事件时，又相对应的创建`clientfd`并仍然放入当前进程的`eventpoll`中，这样当前这条进程就可以同时处理多个套接字了。
单个进程的处理方式无法发挥 CPU 的多核性能，而且在现在的网络流量环境中，单个进程的处理性能也存在瓶颈，因此需要使用多进程的方式。注意这里的多进程是固定的数量，往往是服务器启动之后就创建的，因此没有动态创建和销毁的性能问题。这时的做法是在一个父进程中使用`socket() + bind() + listen()`，然后`fork`出多个子进程，每个子进程创建自己的`eventpoll`，各自调用`epoll_wait`去监听这个`listenfd`的事件，注意对于所有的子进程来说，这个`listenfd`是同一个套接字。在这种做法下，当`listenfd`有相应可读的事件时，所有进程都会被唤醒，造成惊群问题。根本原因在于他添加到套接字等待队列的方式是`prepare_to_wait`的方式，即非互斥式的。
在没有`SO_REUSEPORT`标识和`EPOLLEXCLUSIVE`标识之前，要处理这个问题，只能采用手动加锁的形式。以`nginx`的处理措施为例：它利用了一把进程锁，每个进程在监听这个`listenfd`之前都会先尝试获取这把锁，如果获取成功则把这个`listenfd`放到自己的`eventpoll`中，并设置超时等待时间，在这段时间内如果成功的获取到事件则处理，否则时间到达之后则将其从自己的`eventpoll`中删除，并释放锁。
而现在`EPOLL`支持使用`EPOLLEXCLUSIVE`标识，我们在每个进程中都使用`epoll_ctl`为该`listenfd`添加`EPOLLEXCLUSIVE`属性，这样当有事件到来时，只会唤醒其中的一个进程，这样就避免了惊群问题。
而 linux 提供了`SO_REUSEPORT`标识，这个标识的关键作用是允许 **多个 **`**socket**`** 同时监听相同的**`**IP + PORT**`**，当有事件到来时，内核会自动负载均衡的选择唤醒其中一个**`**socket**`**。**因此在使用`SO_REUSEPORT`时，我们的做法变成了创建多个进程，每个进程独立的使用`socket() + bind() + listen() + epoll_create()`，在这个阶段中多个进程所监听的是同一个`IP + PORT`，对于每个进程来说他们的`listenfd`不是同一个套接字，但却是监听了同一个地址和端口。该`IP + PORT`有事件到达时，内核根据负载均衡原则唤醒其中的一个`socket`，即对应监听的`eventpoll`会监听到可读事件，这样就避免了惊群问题。

注意的是，**SO_REUSEPORT** 参数必须在两条 socket 间同时设置，才能绑定成功，同时对于已经处于 **TIME_WAIT** 状态的 socket，绑定也会失败，因此一般同时使用 **SO_REUSEADDR** 和 **SO_REUSEPORT** 参数

```c
// 复用地址和端口号
// SO_REUSEADDR : 用以解决当服务器主动close之后处于TIME_WAIT状态时，重新启动时出现的地址被占用情况
// SO_REUSEPORT : 用以允许多个socket绑定到相同的ip和port，底层会把接收到的数据负载均衡到绑定的多个socket上
// 一般用以在多进程/多线程中提高负载。
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (char*)&on, sizeof(on));
setsockopt(listenfd, SOL_SOCKET, SO_REUSEPORT, (char*)&on, sizeof(on));
```

#### connect 报错
在上面的描述中，我们是服务端的角色，而当我们是客户端且开启了 **SO_REUSEADDR** 和 **SO_REUSEPORT** 后，意味着允许同时有多个 socket 拥有相同的 `<protocol>, <src addr>, <src port>`。这在未开启这两个参数时，`bind()`阶段就已经报错了。
一般来说，这种方式是比较少的，但是如果这样设置之后，在`connet()`阶段，如果这多个`socket`同时去连接一个目标地址和端口，则会由于整个五元组都相同，而造成异常。