---
layout: post
title: TCP-握手&挥手源码解析:第二次握手
date: 2022-9-9 20:02:34 +0800
categories: TCP
tags: tcp 三次握手 握手&挥手源码解析 
author: Rxsi
---

* content
{:toc}

## 第二次握手
要进入第二次握手的处理流程，服务端必须是先接收到了第一次握手包，此时的处理流程如下：
<!--more-->
```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
	//....
	switch (sk->sk_state) {
	case TCP_LISTEN: // 当前处于LISTEN状态
		if (th->ack) // 收到了ACK包，回复RST报文
			return 1;

		if (th->rst) { // 收到了RST重置报文
			SKB_DR_SET(reason, TCP_RESET); 
			goto discard; // 直接丢掉这个包
		}
		if (th->syn) { // 收到了SYN包
			if (th->fin) { // 包含了FIN标志，说明是第二三次挥手报文
				SKB_DR_SET(reason, TCP_FLAGS);
				goto discard; // 这接丢掉这个包
			}
            // 进入这个阶段，说明是第一次握手包
			/* It is possible that we process SYN packets from backlog,
			 * so we need to make sure to disable BH and RCU right there.
			 */
			rcu_read_lock();
			local_bh_disable();
			acceptable = icsk->icsk_af_ops->conn_request(sk, skb) >= 0; // 因为是IP4，因此对应的方法是tcp_v4_conn_request
			local_bh_enable();
			rcu_read_unlock();

			if (!acceptable) // 如果这个包不可用，返回RST
				return 1;
			consume_skb(skb);
			return 0;
		}
		SKB_DR_SET(reason, TCP_FLAGS);
		goto discard;
    }
}
```
### tcp_v4_conn_request
这个函数是处理第一次握手包之前的前置函数，用以对广播的数据包进行忽略。
```c
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
	/* Never answer to SYNs send to broadcast or multicast */
	if (skb_rtable(skb)->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST)) // 不回复广播的包
		goto drop;

	return tcp_conn_request(&tcp_request_sock_ops,
				&tcp_request_sock_ipv4_ops, sk, skb); // 进入接收处理阶段

drop:
	tcp_listendrop(sk);
	return 0;
}
```
### tcp_conn_request
```c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_fastopen_cookie foc = { .len = -1 };
	__u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn; // 这是服务端的序列号，如果在这里获取到的是空，那么后面会被初始化
	struct tcp_options_received tmp_opt; // 这个包含的是从数据包中解析出来的数据
	struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct sock *fastopen_sk = NULL;
	struct request_sock *req;
	bool want_cookie = false;
	struct dst_entry *dst;
	struct flowi fl;
	u8 syncookies;

	syncookies = READ_ONCE(net->ipv4.sysctl_tcp_syncookies); // 读取syn_cookie配置，有三种可能：0：不开启；1：只有压力大时才会启用；2：始终开启

	/* TW buckets are converted to open requests without
	 * limitations, they conserve resources and peer is
	 * evidently real one.
	 */
	if ((syncookies == 2 || inet_csk_reqsk_queue_is_full(sk)) && !isn) { // 如果是始终开启或者icsk_accept_queue的容量已经大于sk_max_ack_backlog配置，即半连接队列满了
		want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name); // 再根据syncookies配置看是否能够发送cookie，如果不行，则会丢掉这个包，这意味着链接失败。这里只要配置是1或者2都会返回true，所以如果半连接满了，而配置是0那么此处就会直接丢掉这个包。
		if (!want_cookie)
			goto drop;
	}

	if (sk_acceptq_is_full(sk)) { // 如果全连接队列满了，也是不能再接收新数据包，因此直接丢掉
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}

	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie); // 申请request_sock结构体，之所以不直接使用sock结构体是因为request_sock更轻量，而当遇到flood攻击时，只是创建request_sock能够保证更小的开支
	if (!req) // 创建失败则丢掉这个包
		goto drop;
    // 对request_sock的数据填充
	// ... 
    if (!want_cookie && !isn) { // 如果没有使用syncookie，并且服务端的序列号还没有确定
		int max_syn_backlog = READ_ONCE(net->ipv4.sysctl_max_syn_backlog); // 读取配置的最大backlog值

		/* Kill the following clause, if you dislike this way. */
		if (!syncookies &&
		    (max_syn_backlog - inet_csk_reqsk_queue_len(sk) <
		     (max_syn_backlog >> 2)) &&
		    !tcp_peer_is_proven(req, dst)) { // 如果没有使用syncookie且半连接队列剩余的空间只剩下1/4，那么直接丢掉这个包。这是因为在没有开启半连接队列时，会把1/4的容量用来存储目标信息？？不是很理解这个用意。。。。TODO
			/* Without syncookies last quarter of
			 * backlog is filled with destinations,
			 * proven to be alive.
			 * It means that we continue to communicate
			 * to destinations, already remembered
			 * to the moment of synflood.
			 */
			pr_drop_req(req, ntohs(tcp_hdr(skb)->source),
				    rsk_ops->family);
			goto drop_and_release;
		}

		isn = af_ops->init_seq(skb); // 调用tcp专属的request_sock_ops，在这里是tcp_v4_init_seq方法，实际调用的就是和客户端在调用connect时所用的计算序列号的方式是一样的.
	}

	tcp_ecn_create_request(req, skb, sk, dst);

	if (want_cookie) {
		isn = cookie_init_sequence(af_ops, sk, skb, &req->mss); // 如果使用了cookie，那么序列号会变成cookie值。计算函数是cookie_v4_init_sequence
		if (!tmp_opt.tstamp_ok)
			inet_rsk(req)->ecn_ok = 0;
	}

	tcp_rsk(req)->snt_isn = isn; // 序列号
	tcp_rsk(req)->txhash = net_tx_rndhash();
	tcp_rsk(req)->syn_tos = TCP_SKB_CB(skb)->ip_dsfield;
	tcp_openreq_init_rwin(req, sk, dst);
	sk_rx_queue_set(req_to_sk(req), skb); // 把sk_buff 放入到sk里面的红黑树
	if (!want_cookie) {
		tcp_reqsk_record_syn(sk, req, skb);
		fastopen_sk = tcp_try_fastopen(sk, skb, req, &foc, dst); // 是否开启了fastopen
	}
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, TCP_SYNACK_FASTOPEN, skb); // 如果开启了fastopen，则传入TCP_SYNACK_FASTOPEN标志，用以说明本次可以连带发送数据
		/* Add the child socket directly into the accept queue */
		if (!inet_csk_reqsk_queue_add(sk, req, fastopen_sk)) { // 加入半连接队列，添加失败就丢掉这个包
			reqsk_fastopen_remove(fastopen_sk, req, false);
			bh_unlock_sock(fastopen_sk);
			sock_put(fastopen_sk);
			goto drop_and_free;
		}
		sk->sk_data_ready(sk);
		bh_unlock_sock(fastopen_sk);
		sock_put(fastopen_sk);
	} else {
		tcp_rsk(req)->tfo_listener = false;
		if (!want_cookie) {
			req->timeout = tcp_timeout_init((struct sock *)req); // 如果没有fastopen且没有使用syncookies，则插入半连接队列并设置超时时间，会进行定时的重传
			inet_csk_reqsk_queue_hash_add(sk, req, req->timeout);
		}
		af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE,
				    skb); // 发送SYN-ACK包
    	// ......
	}
}
```
是否开启`syncookie`由以下配置决定：
```shell
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_syncookies 
1
```
默认的值为 **1**，代表了只在半连接队列满了之后才使用 cookie；**2 **代表的是总是开启 cookie 功能，**0 **代表不开启 cookie 功能，此时如果半连接队列满了，那么会直接丢掉这个`SYN`包。
当使用了 cookie 功能时，服务端发送给客户端的序列号就是计算出的 cookie 值，TODO ？怎么确定服务端的发送序列号
定时重发次数由`tcp_synack_retries`决定：
```shell
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_synack_retries 
5
```
### tcp_v4_send_synack
如果是 IPV4 网络，那么上面的`send_synack`函数就是`tcp_v4_send_synack`方法，实现如下：
```c
static int tcp_v4_send_synack(const struct sock *sk, struct dst_entry *dst,
			      struct flowi *fl,
			      struct request_sock *req,
			      struct tcp_fastopen_cookie *foc,
			      enum tcp_synack_type synack_type,
			      struct sk_buff *syn_skb)
{
	const struct inet_request_sock *ireq = inet_rsk(req);
	struct flowi4 fl4;
	int err = -1;
	struct sk_buff *skb;
	u8 tos;

	/* First, grab a route. */
	if (!dst && (dst = inet_csk_route_req(sk, &fl4, req)) == NULL) // 如果传进来的dst目标地址是空的，那么就从路由中查找，如果查找失败则直接退出
		return -1;

	skb = tcp_make_synack(sk, dst, req, foc, synack_type, syn_skb); // 创建SYN-ACK包

	if (skb) {
		__tcp_v4_send_check(skb, ireq->ir_loc_addr, ireq->ir_rmt_addr); // 计算checksum校验码

		tos = READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_reflect_tos) ?
				(tcp_rsk(req)->syn_tos & ~INET_ECN_MASK) |
				(inet_sk(sk)->tos & INET_ECN_MASK) :
				inet_sk(sk)->tos;

		if (!INET_ECN_is_capable(tos) &&
		    tcp_bpf_ca_needs_ecn((struct sock *)req))
			tos |= INET_ECN_ECT_0;

		rcu_read_lock();
		err = ip_build_and_send_pkt(skb, sk, ireq->ir_loc_addr,
					    ireq->ir_rmt_addr,
					    rcu_dereference(ireq->ireq_opt),
					    tos); // 生成IP数据包并发送
		rcu_read_unlock();
		err = net_xmit_eval(err);
	}

	return err;
}
```
### tcp_make_synack
该函数用以构建`SYN-ACK`包
```c
struct sk_buff *tcp_make_synack(const struct sock *sk, struct dst_entry *dst,
				struct request_sock *req,
				struct tcp_fastopen_cookie *foc,
				enum tcp_synack_type synack_type,
				struct sk_buff *syn_skb)
{
	struct inet_request_sock *ireq = inet_rsk(req);
	const struct tcp_sock *tp = tcp_sk(sk);
	struct tcp_md5sig_key *md5 = NULL;
	struct tcp_out_options opts;
	struct sk_buff *skb;
	int tcp_header_size;
	struct tcphdr *th;
	int mss;
	u64 now;

	skb = alloc_skb(MAX_TCP_HEADER, GFP_ATOMIC); // 申请内存空间
	if (unlikely(!skb)) {
		dst_release(dst);
		return NULL;
	}
	/* Reserve space for headers. */
	skb_reserve(skb, MAX_TCP_HEADER); // 为MAC层、IP层、TCP层预留足够的头部空间

	switch (synack_type) { // 这里的目的是把skb和sk绑定在一起
	case TCP_SYNACK_NORMAL: // 普通的包
		skb_set_owner_w(skb, req_to_sk(req));
		break;
	case TCP_SYNACK_COOKIE: // 开启了syncookie
		/* Under synflood, we do not attach skb to a socket,
		 * to avoid false sharing.
		 */
		break;
	case TCP_SYNACK_FASTOPEN: // 开启了fastopen，，可携带数据
		/* sk is a const pointer, because we want to express multiple
		 * cpu might call us concurrently.
		 * sk->sk_wmem_alloc in an atomic, we can promote to rw.
		 */
		skb_set_owner_w(skb, (struct sock *)sk);
		break;
	}
	skb_dst_set(skb, dst); // 设置skb的发送目标信息

	mss = tcp_mss_clamp(tp, dst_metric_advmss(dst)); // 设置mss

	memset(&opts, 0, sizeof(opts)); // 申请TCP输出包的内存
	now = tcp_clock_ns(); 
#ifdef CONFIG_SYN_COOKIES // 时间戳相关的设置
	if (unlikely(synack_type == TCP_SYNACK_COOKIE && ireq->tstamp_ok))
		skb_set_delivery_time(skb, cookie_init_timestamp(req, now),
				      true);
	else
#endif
	{
		skb_set_delivery_time(skb, now, true);
		if (!tcp_rsk(req)->snt_synack) /* Timestamp first SYNACK */
			tcp_rsk(req)->snt_synack = tcp_skb_timestamp_us(skb);
	}

#ifdef CONFIG_TCP_MD5SIG // md5值的设置
	rcu_read_lock();
	md5 = tcp_rsk(req)->af_specific->req_md5_lookup(sk, req_to_sk(req));
#endif
	skb_set_hash(skb, tcp_rsk(req)->txhash, PKT_HASH_TYPE_L4);
	/* bpf program will be interested in the tcp_flags */
	TCP_SKB_CB(skb)->tcp_flags = TCPHDR_SYN | TCPHDR_ACK;
	tcp_header_size = tcp_synack_options(sk, req, mss, skb, &opts, md5,
					     foc, synack_type,
					     syn_skb) + sizeof(*th); // 计算实际TCP header需要的空间，在上面我们申请的头空间是TCP header的最大空间

	skb_push(skb, tcp_header_size);
	skb_reset_transport_header(skb);

	th = (struct tcphdr *)skb->data;
	memset(th, 0, sizeof(struct tcphdr));
	th->syn = 1; // syn标志
	th->ack = 1; // ack标志
	tcp_ecn_make_synack(req, th);
	th->source = htons(ireq->ir_num);
	th->dest = ireq->ir_rmt_port;
	skb->mark = ireq->ir_mark;
	skb->ip_summed = CHECKSUM_PARTIAL;
	th->seq = htonl(tcp_rsk(req)->snt_isn); // 服务端的序列号，如果使用了syncookie功能，那么序列号就是cookie值。
	/* XXX data is queued and acked as is. No buffer/window check */
	th->ack_seq = htonl(tcp_rsk(req)->rcv_nxt); // 回复客户端的序列号

	/* RFC1323: The window in SYN & SYN/ACK segments is never scaled. */
	th->window = htons(min(req->rsk_rcv_wnd, 65535U));
	tcp_options_write(th, NULL, &opts);
	th->doff = (tcp_header_size >> 2);
	__TCP_INC_STATS(sock_net(sk), TCP_MIB_OUTSEGS);

#ifdef CONFIG_TCP_MD5SIG
	/* Okay, we have all we need - do the md5 hash if needed */
	if (md5)
		tcp_rsk(req)->af_specific->calc_md5_hash(opts.hash_location,
					       md5, req_to_sk(req), skb);
	rcu_read_unlock();
#endif

	bpf_skops_write_hdr_opt((struct sock *)sk, skb, req, syn_skb,
				synack_type, &opts);

	skb_set_delivery_time(skb, now, true);
	tcp_add_tx_delay(skb, tp);

	return skb;
}
```
### 第二次握手丢失，会发生什么？
由于客户端接收不到`SYN-ACK`报文，因此客户端会认为由它发出的第一次握手包丢失了，此时就会进行`SYN`包的重发，由配置`tcp_syn_retries（默认6）`决定最大重传次数。而由于服务端是已经接收到了之前的`SYN`包，即已经处于`SYN_RECV`状态，那么在收到重复的`SYN`报文时会就会立即重发`SYN-ACK`报文，并重置`SYN-ACK`的重传次数
```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
    // ...
    req = rcu_dereference_protected(tp->fastopen_rsk,
					lockdep_sock_is_held(sk)); // 获取request_sock结构
	if (req) {
		bool req_stolen;

		WARN_ON_ONCE(sk->sk_state != TCP_SYN_RECV &&
		    sk->sk_state != TCP_FIN_WAIT1);

		if (!tcp_check_req(sk, skb, req, true, &req_stolen)) { // 检查该包是否可接收
			SKB_DR_SET(reason, TCP_FASTOPEN);
			goto discard;
		}
	}
    // ....
}

struct sock *tcp_check_req(struct sock *sk, struct sk_buff *skb,
			   struct request_sock *req,
			   bool fastopen, bool *req_stolen)
{
    // .....
	// 收到了SYN包
	/* Check for pure retransmitted SYN. */
	if (TCP_SKB_CB(skb)->seq == tcp_rsk(req)->rcv_isn &&
	    flg == TCP_FLAG_SYN &&
	    !paws_reject) {
		if (!tcp_oow_rate_limited(sock_net(sk), skb,
					  LINUX_MIB_TCPACKSKIPPEDSYNRECV,
					  &tcp_rsk(req)->last_oow_ack_time) &&

		    !inet_rtx_syn_ack(sk, req)) {
			unsigned long expires = jiffies;

			expires += reqsk_=timeout(req, TCP_RTO_MAX);
			if (!fastopen)
				mod_timer_pending(&req->rsk_timer, expires);
			else
				req->rsk_timer.expires = expires;
		}
		return NULL; // 这里返回 NULL
	}

```
而服务端本身也会对`SYN-ACK`报文进行定时重发，即如果客户端没能在规定的时间内回复`ACK`报文，那么服务端就会自动进行`SYN-ACK`报文的重发，最大重传次数由`tcp_synack_retries（默认5）`决定。