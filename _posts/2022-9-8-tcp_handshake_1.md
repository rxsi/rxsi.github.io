---
layout: post
title: 握手&挥手源码解析:第一次握手
date: 2022-9-8 20:28:03 +0800
categories: TCP
tags: 三次握手 源码分析
author: Rxsi
---

* content
{:toc}

## 第一次握手
### tcp_v4_connect
在`IPV4`的条件下，第一次握手对应的内核源码是`tcp_v4_connect`，具体代码内容如下：
<!--more-->
```c
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    struct sockaddr_in *usin = (struct sockaddr_in *)uaddr; // 上层传入的目标地址信息
    struct inet_sock *inet = inet_sk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    __be16 orig_sport, orig_dport; // __be16、__be32是大端序，网络序是大端序
    __be32 daddr, nexthop;
    struct flowi4 *fl4;
    struct rtable *rt;
    int err;
    struct ip_options_rcu *inet_opt;
    struct inet_timewait_death_row *tcp_death_row = sock_net(sk)->ipv4.tcp_death_row;

    if (addr_len < sizeof(struct sockaddr_in)) // 地址长度校验
        return -EINVAL;

    if (usin->sin_family != AF_INET) // 是否是IPV4
        return -EAFNOSUPPORT;

    // 一些地址、路由信息的处理
    // ....
    
    tp->rx_opt.mss_clamp = TCP_MSS_DEFAULT; // 设置报文的初始MSS值, 默认值为563

    tcp_set_state(sk, TCP_SYN_SENT); // 设置状态为 SYN_SENT
    err = inet_hash_connect(tcp_death_row, sk); // 计算客户端的端口，是通过根据目标IP和端口计算出一个随机数，然后从这个范围里面逐个查找可用的端口号。这里面涉及到tcp_tw_reuse选项，如果设置了该选项，那么就可以复用处于tw状态的socket了。
    if (err)
        goto failure; // 绑定端口号失败
    
    // 地址路由的处理
    // ......

    if (likely(!tp->repair)) {
        if (!tp->write_seq) // 没有设置初始的序号
            WRITE_ONCE(tp->write_seq,
                   secure_tcp_seq(inet->inet_saddr,
                          inet->inet_daddr,
                          inet->inet_sport,
                          usin->sin_port)); // 计算规则是根据双方地址+端口计算出一个seq值，然后再加上(纳秒级时间戳>>6)得到的值，这个值是唯一的。
        tp->tsoffset = secure_tcp_ts_off(sock_net(sk),
                         inet->inet_saddr,
                         inet->inet_daddr);
    }
    
    // ....
    
    err = tcp_connect(sk); // 真正构造SYN包并发送

    if (err)
        goto failure;

    return 0;

failure: // 如果上面有任何一项校验失败，则跳转到这里
    tcp_set_state(sk, TCP_CLOSE); // 设置状态为 CLOSE
    ip_rt_put(rt);
    sk->sk_route_caps = 0;
    inet->inet_dport = 0;
    return err;
}
```
可见`tcp_v4_connect`负责 IP 地址和路由信息的校验与设置，同时会设置 TCP 状态并初始发送序列号
###  tcp_connect
经过了`tcp_v4_connect`的必要设置，在`tcp_connect`会进行`SYN`包的构建与发送
```c
/* Build a SYN and send it off. */
int tcp_connect(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk); // 将sock结构体转换成tcp_sock结构体。tcp_sock结构体包含了sock结构体，同时包含了更多tcp独有的数据项
    struct sk_buff *buff; // sk_buff是一个数据报的数据集合，会以双向链表和红黑树进行组织管理，这个和VMA的管理方式一致。
    int err;

    tcp_call_bpf(sk, BPF_SOCK_OPS_TCP_CONNECT_CB, 0, NULL);

    if (inet_csk(sk)->icsk_af_ops->rebuild_header(sk))
        return -EHOSTUNREACH; /* Routing failure or similar. */

    tcp_connect_init(sk); // 初始化tcp_sock里面的TCP信息，包括初始发送/接收序列号，发送/接收窗口、初始化rto=1.

    if (unlikely(tp->repair)) {
        tcp_finish_connect(sk, NULL);
        return 0;
    }

    buff = tcp_stream_alloc_skb(sk, 0, sk->sk_allocation, true); // 申请一个sk_buff
    if (unlikely(!buff))
        return -ENOBUFS;

    tcp_init_nondata_skb(buff, tp->write_seq++, TCPHDR_SYN); // sk_buff初始化
    tcp_mstamp_refresh(tp); // 设置时间戳信息
    tp->retrans_stamp = tcp_time_stamp(tp);
    tcp_connect_queue_skb(sk, buff);
    tcp_ecn_send_syn(sk, buff);
    tcp_rbtree_insert(&sk->tcp_rtx_queue, buff); // 将sk_buff插入sk里面的红黑树中，后续要获取sk_buff只需要从sk->tcp_rtx_queue这颗红黑树上查找即可。

    /* Send off SYN; include data in Fast Open. */
    err = tp->fastopen_req ? tcp_send_syn_data(sk, buff) :
          tcp_transmit_skb(sk, buff, 1, sk->sk_allocation); // 如果打开了fast open选项，意味着可以在发送SYN包时发送数据，那么后续调用tcp_send_syn_data方法，否则调用tcp_transmit_skb方法，而实际在tcp_send_syn_data中最终调用的也是tcp_transmit_skb方法。tcp_transmit_skb方法负责的是将数据包（sk_buff）发送到IP层
    if (err == -ECONNREFUSED)
        return err;

    /* We change tp->snd_nxt after the tcp_transmit_skb() call
     * in order to make this packet get counted in tcpOutSegs.
     */
    WRITE_ONCE(tp->snd_nxt, tp->write_seq); // 更新序列号
    tp->pushed_seq = tp->write_seq;
    buff = tcp_send_head(sk);
    if (unlikely(buff)) {
        WRITE_ONCE(tp->snd_nxt, TCP_SKB_CB(buff)->seq);
        tp->pushed_seq	= TCP_SKB_CB(buff)->seq;
    }
    TCP_INC_STATS(sock_net(sk), TCP_MIB_ACTIVEOPENS);

    /* Timer for repeating the SYN until an answer. */
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                  inet_csk(sk)->icsk_rto, TCP_RTO_MAX); // 超时重传定时器，inet_csk(sk)->icsk_rto值为1,在前面tcp_connect_init中初始化的。后续在定时器超时之后，会以 icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX) 方式进行翻倍增长，最大是120。每次定时器到达时会判断当前已经重试的次数，达到tcp_syn_retries配置次数之后就会停止定时器
    return 0;
}
```
可以看到在这个函数内，会构建`SYN`包，并将其传递给更下层的 IP 层进行发包，同时也会启动超时重传定时器，以初始超时时间为1，后续每秒都倍增。最大的重试次数是`tcp_syn_retries`，默认值为 **6**。
```shell
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_syn_retries 
6
```
### tcp_retransmit_timer
该函数负责处理所有需要重发的数据包
```c
void tcp_retransmit_timer(struct sock *sk)
{
    // .....
    if (!tp->snd_wnd && !sock_flag(sk, SOCK_DEAD) &&
        !((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV))) { // 当对方接收窗口已经变为0的处理
    // ....
    }

    if (tcp_write_timeout(sk)) // 判断是否已经达到重传次数的上限，如果达到上限，那么在这个函数里面会关闭tcp，并返回1
        goto out;

    icsk->icsk_retransmits++; // 重传次数递增
    if (tcp_retransmit_skb(sk, tcp_rtx_queue_head(sk), 1) > 0) { // 进行重传，内部调用的是tcp_transmit_skb方法，如果返回>0代表重传失败，意味着当前系统资源不足。则直接进行下一轮的定时器，进行重试，定时器的时间戳为TCP_RESOURCE_PROBE_INTERVAL
        inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                      TCP_RESOURCE_PROBE_INTERVAL,
                      TCP_RTO_MAX);
        goto out;
    }

out_reset_timer:
    if (sk->sk_state == TCP_ESTABLISHED &&
        (tp->thin_lto || READ_ONCE(net->ipv4.sysctl_tcp_thin_linear_timeouts)) &&
        tcp_stream_is_thin(tp) &&
        icsk->icsk_retransmits <= TCP_THIN_LINEAR_RETRIES) { // 链接已经建立时的重传
        icsk->icsk_backoff = 0;
        icsk->icsk_rto = min(__tcp_set_rto(tp), TCP_RTO_MAX); // 重传时间根据rtt计算
    } else {
        /* Use normal (exponential) backoff */
        icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX); // 其他状态的数据包的重传时间都翻倍增长
    }
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                  tcp_clamp_rto_to_user_timeout(sk), TCP_RTO_MAX); // 开启新的定时器，时间间隔是icsk_rto
    if (retransmits_timed_out(sk, READ_ONCE(net->ipv4.sysctl_tcp_retries1) + 1, 0))
        __sk_dst_reset(sk);

out:;
}

static int tcp_write_timeout(struct sock *sk)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct tcp_sock *tp = tcp_sk(sk);
    struct net *net = sock_net(sk);
    bool expired = false, do_reset;
    int retry_until;

    if ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) { // 当前处于SYN_SENT或者SYN_RECV状态
        if (icsk->icsk_retransmits)
            __dst_negative_advice(sk);
        retry_until = icsk->icsk_syn_retries ? :
            READ_ONCE(net->ipv4.sysctl_tcp_syn_retries);
        expired = icsk->icsk_retransmits >= retry_until; // 判断已重传次数是否已经达到sysctl_tcp_syn_retries配置
    } 
    // ......
    if (expired) { // 已经达到重传上限
        /* Has it gone just too far? */
        tcp_write_err(sk); // 关闭
        return 1;
    }
    // ......
    return 0; // 定时器还可以继续
}
```
### 第一次握手丢失会发生什么？
当客户端发送完`SYN`报文，那么就会进入`SYN_SEND`状态，在这之后如果未能收到服务端回复的`SYN-ACK`报文，那么就会通过超时重传机制进行重发，最大的重发次数由`tcp_syn_retries（默认是6）`控制。如果在限定的次数内没有成功接收到第二次握手包，那么就会关闭 TCP，转为`CLOSE`状态。

在 TCP 关闭之后如果接收到了上一次的第二次握手包，由于该包是 TCP 报文，因此 IP 层会调用`tcp_v4_rcv`方法进行处理。而此时如果客户端没有在监听这个端口，那么会由于无法查找到对应的 TCP socket，会回复`RST`报文。
```c
// IP层接收到TCP数据包时会调用这个方法
int tcp_v4_rcv(struct sk_buff *skb)
{
    // 忽略一些包体长度校验
lookup:
    sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
                   th->dest, sdif, &refcounted); // 这里传进的是source端口和dest端口，在里面会获取source的IP和dest的IP，然后再进行查找
    if (!sk) // 没有查找到socket，跳转到no_tcp_socket
        goto no_tcp_socket;
    // ....
no_tcp_socket:
    drop_reason = SKB_DROP_REASON_NO_SOCKET;
    if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) // 这里固定返回1，即!1得到的结果是false
        goto discard_it;

    tcp_v4_fill_cb(skb, iph, th);

    if (tcp_checksum_complete(skb)) { // 检查check_sum失败
csum_error:
        drop_reason = SKB_DROP_REASON_TCP_CSUM;
        trace_tcp_bad_csum(skb);
        __TCP_INC_STATS(net, TCP_MIB_CSUMERRORS);
bad_packet:
        __TCP_INC_STATS(net, TCP_MIB_INERRS);
    } else { // check_sum校验成功，回复RST报文。
        tcp_v4_send_reset(NULL, skb);
    }
    // ..........
}
```
通过上面的源码我们可以知道，只要**五元组**对应不上，那么就无法成功找到 socket，系统就会回复`RST`报文。

而如果在客户端这边，正好有一个**五元组**对应上，那么后续的处理逻辑会由于**序列号**对应不上，而回复`RST`报文
```c
// TCP处于SYN_SENT状态时的处理函数
static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb,
                     const struct tcphdr *th)
{
    // .....
    if (th->ack) { // 像下面的注释一样，如果这个包有ACK标志，并且序列号范围不对，那么会返回RST包。这种情况就处理了旧的第二次握手包的问题
        /* rfc793:
         * "If the state is SYN-SENT then
         *    first check the ACK bit
         *      If the ACK bit is set
         *	  If SEG.ACK =< ISS, or SEG.ACK > SND.NXT, send
         *        a reset (unless the RST bit is set, if so drop
         *        the segment and return)"
         */
        if (!after(TCP_SKB_CB(skb)->ack_seq, tp->snd_una) ||
            after(TCP_SKB_CB(skb)->ack_seq, tp->snd_nxt)) {
            /* Previous FIN/ACK or RST/ACK might be ignored. */
            if (icsk->icsk_retransmits == 0)
                inet_csk_reset_xmit_timer(sk,
                        ICSK_TIME_RETRANS,
                        TCP_TIMEOUT_MIN, TCP_RTO_MAX);
            goto reset_and_undo; // 这里会返回1，并在上层被捕获，然后发送RST包
        }
    }


reset_and_undo:
    tcp_clear_options(&tp->rx_opt);
    tp->rx_opt.mss_clamp = saved_clamp;
    return 1; // 上层会捕获这个值
}
```
客户端如果在关闭旧socket之后立即重新发送第一次握手，如果随机出的新端口和上一次的不一样，那么服务端会新建一个 socket 与之对应，此时服务端会成功进入`SYN_RECV`状态。如果在这之前服务端已经根据旧的握手包建立了一个 socket，那么该socket 就会因为定时重发了`SYN-ACK`包而达到次数上限或者收到了客户端返回的`RST`报文而销毁。

如果新端口和上一次的端口是一样的，且服务端因为接收了上一次的旧握手包已处于`SYN_RECV`状态，则服务端会返回的`SYN-ACK`。由于里面包含的是序列号是上一次握手的，因此在客户端接收到之后会返回`RST`使服务端进行重置，后续会继续重发新握手包，建立新链接。

相对应的，如果新的握手包先于旧的握手包到达了服务端，那么服务端接收到旧的握手包也是通过`SYN-ACK`返回当前记录的正确的序列号，而客户端接收到之后不会因此而关闭连接，只会再次重发序号正确的数据包。相较于旧的握手包先到的处理流程，**这时候其实客户端就起到了协助判断是否需要重置当前连接的作用，这也是三次握手的必要性！**

```c
struct sock *tcp_check_req(struct sock *sk, struct sk_buff *skb,
               struct request_sock *req,
               bool fastopen, bool *req_stolen)
{
    // ....
    if (paws_reject || !tcp_in_window(TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq,
                  tcp_rsk(req)->rcv_nxt, tcp_rsk(req)->rcv_nxt + req->rsk_rcv_wnd)) { // 处于SYN_RECV状态的服务端接收到了一个不在窗口内的包
    /* Out of window: send ACK and drop. */
    if (!(flg & TCP_FLAG_RST) &&
        !tcp_oow_rate_limited(sock_net(sk), skb,
                  LINUX_MIB_TCPACKSKIPPEDSYNRECV,
                  &tcp_rsk(req)->last_oow_ack_time))
        req->rsk_ops->send_ack(sk, skb, req); // 返回ACK报文，服务端本身建立的连接有可能是旧的，因此返回ACK报文，由客户端协助判断。
    if (paws_reject)
        __NET_INC_STATS(sock_net(sk), LINUX_MIB_PAWSESTABREJECTED);
    return NULL;
    }
}    
```