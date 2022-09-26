---
layout: post
title: TCP-握手&挥手源码解析:第三次握手
date: 2022-9-10 19:32:44 +0800
categories: TCP
tags: tcp 三次握手 握手&挥手源码解析 
author: Rxsi
---

* content
{:toc}

## 第三次握手
### tcp_rcv_synsent_state_processy
当客户端接收到了第二次握手包后，就进入第三次握手判断阶段，此时客户端处于`SYN_SENT`阶段
<!--more-->
```c
// 当TCP处于SYN_SENT状态时的处理函数
static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb,
					 const struct tcphdr *th)
{
    // ...
	if (th->ack) {
		/* rfc793:
		 * "If the state is SYN-SENT then
		 *    first check the ACK bit
		 *      If the ACK bit is set
		 *	  If SEG.ACK =< ISS, or SEG.ACK > SND.NXT, send
		 *        a reset (unless the RST bit is set, if so drop
		 *        the segment and return)"
		 */
		if (!after(TCP_SKB_CB(skb)->ack_seq, tp->snd_una) ||
		    after(TCP_SKB_CB(skb)->ack_seq, tp->snd_nxt)) { // 收到了不符合区间的含有ACK标志的包，回复一个RST报文
			/* Previous FIN/ACK or RST/ACK might be ignored. */
			if (icsk->icsk_retransmits == 0)
				inet_csk_reset_xmit_timer(sk,
						ICSK_TIME_RETRANS,
						TCP_TIMEOUT_MIN, TCP_RTO_MAX);
			goto reset_and_undo; // 返回RST报文或者直接丢掉这个包
		}

		if (tp->rx_opt.saw_tstamp && tp->rx_opt.rcv_tsecr &&
		    !between(tp->rx_opt.rcv_tsecr, tp->retrans_stamp,
			     tcp_time_stamp(tp))) { // 在可接收的时间范围内，这是为了避免接收已经过期的数据包
			NET_INC_STATS(sock_net(sk),
					LINUX_MIB_PAWSACTIVEREJECTED);
			goto reset_and_undo;
		}

		/* Now ACK is acceptable.
		 *
		 * "If the RST bit is set
		 *    If the ACK was acceptable then signal the user "error:
		 *    connection reset", drop the segment, enter CLOSED state,
		 *    delete TCB, and return."
		 */

		if (th->rst) { // 如果收到了一个符合区间和时间内的RST+ACK报文，那么就转为CLOSED状态，并且直接丢弃这个包，不做回复
			tcp_reset(sk, skb);
consume:
			__kfree_skb(skb);
			return 0;
		}

		/* rfc793:
		 *   "fifth, if neither of the SYN or RST bits is set then
		 *    drop the segment and return."
		 *
		 *    See note below!
		 *                                        --ANK(990513)
		 */
		if (!th->syn) { // 既不含RST标志也不含SYN标志，只含有ACK标志，那么丢掉这个包，因为本阶段并不能处理这个包
			SKB_DR_SET(reason, TCP_FLAGS);
			goto discard_and_undo;
		}
        // ....
        tcp_ack(sk, skb, FLAG_SLOWPATH); // 对ACK包进行解析，更新DACK、FACK等信息
    	// ....
		tp->snd_wnd = ntohs(th->window); // 注意，接收到SYN或者SYN-ACK包，是不对窗口进行缩放处理的

		if (!tp->rx_opt.wscale_ok) { // 使缩放不生效
			tp->rx_opt.snd_wscale = tp->rx_opt.rcv_wscale = 0;
			tp->window_clamp = min(tp->window_clamp, 65535U);
		}
    	// ......
		tcp_finish_connect(sk, skb); // 设置连接成功的状态

		fastopen_fail = (tp->syn_fastopen || tp->syn_data) &&
				tcp_rcv_fastopen_synack(sk, skb, &foc); // 如果开启了fastopen，那么使用tcp_rcv_fastopen_synack处理包中的数据
    	// .....
		if (sk->sk_write_pending ||
		    icsk->icsk_accept_queue.rskq_defer_accept ||
		    inet_csk_in_pingpong_mode(sk)) { // 如果开启了延迟确认机制，注意这个机制会延迟ACK的回复，所以可能会造成3次挥手
			/* Save one ACK. Data will be ready after
			 * several ticks, if write_pending is set.
			 *
			 * It may be deleted, but with this feature tcpdumps
			 * look so _wonderfully_ clever, that I was not able
			 * to stand against the temptation 8)     --ANK
			 */
			inet_csk_schedule_ack(sk);
			tcp_enter_quickack_mode(sk, TCP_MAX_QUICKACKS);
			inet_csk_reset_xmit_timer(sk, ICSK_TIME_DACK,
						  TCP_DELACK_MAX, TCP_RTO_MAX); // 开启ACK发送定时器
			goto consume;
		}
		tcp_send_ack(sk); // 没有开启延迟ACK，那么立即回复ACK报文
		return -1;
	}
    // ......
}
```
### tcp_finish_connect
此函数负责将 socket 状态设置为`ESTABLISHED`状态
```c
void tcp_finish_connect(struct sock *sk, struct sk_buff *skb)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);

	tcp_set_state(sk, TCP_ESTABLISHED); // 设置为ESTABLISHED状态
	icsk->icsk_ack.lrcvtime = tcp_jiffies32;

	if (skb) {
		icsk->icsk_af_ops->sk_rx_dst_set(sk, skb);
		security_inet_conn_established(sk, skb);
		sk_mark_napi_id(sk, skb);
	}

	tcp_init_transfer(sk, BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB, skb);

	/* Prevent spurious tcp_cwnd_restart() on first data
	 * packet.
	 */
	tp->lsndtime = tcp_jiffies32;

	if (sock_flag(sk, SOCK_KEEPOPEN)) // 开启keepavlie保活机制
		inet_csk_reset_keepalive_timer(sk, keepalive_time_when(tp));

	if (!tp->rx_opt.snd_wscale)
		__tcp_fast_path_on(tp, tp->snd_wnd);
	else
		tp->pred_flags = 0;
}
```
当 TCP 空闲时长达到`tcp_keepalive_time（默认2小时）`后，将会开始进行探测，总探测次数为`tcp_keepalive_probes（默认9次）`，每次的探测间隔为`tcp_keepalive_intvl（默认75秒）`，因此在实际应用中，我们需要在应用层再自行添加一个保活机制。此外，单纯依赖于 TCP 层的保活机制并不能准确的反应出当前应用层的状态，比如应用层出于某些原因负载过高、CPU无法处理任务、线程池满了等，无法对业务进行响应，此时依赖于 TCP 自身是无法发现异常的，因此自行在应用层实现保活机制是有必要的。
```c
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_keepalive_time 
7200
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_keepalive_intvl 
75
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_keepalive_probes 
9
```
回调函数是`tcp_keepalive_timer`，这个回调函数是保活机制和`FIN_WAIT_2`定时器共用
```c
static void tcp_keepalive_timer (struct timer_list *t)
{
	struct sock *sk = from_timer(sk, t, sk_timer);
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	u32 elapsed;

	/* Only process if socket is not in use. */
	bh_lock_sock(sk);
	if (sock_owned_by_user(sk)) { // 如果当前sock正在被使用，这里内部是通过判断sock上的锁拥有者
		/* Try again later. */
		inet_csk_reset_keepalive_timer (sk, HZ/20); // 那么1/20秒之后再次尝试
		goto out;
	}

	if (sk->sk_state == TCP_LISTEN) { // 错误状态
		pr_err("Hmm... keepalive on a LISTEN ???\n");
		goto out;
	}

	tcp_mstamp_refresh(tp);
	if (sk->sk_state == TCP_FIN_WAIT2 && sock_flag(sk, SOCK_DEAD)) { // TCP_FIN_WAIT2状态定时器的处理
		if (tp->linger2 >= 0) { // linger2是通过setsockopt设置的fin_wait2的生命周期，可用以覆盖tcp_fin_timeout配置
			const int tmo = tcp_fin_time(sk) - TCP_TIMEWAIT_LEN; // tcp_fin_time是计算当前fin_wait_2的生命周期，结果是linger2或者tcp_fin_timeout，且最小不能小于 rto << 2 - rot >> 1

			if (tmo > 0) { // 在外层时已经使用 tcp_fin_time - TCP_TIMEWAIT_LEN 计算了差值，到这里已经是跑过了差值时间了，因此这里的计算理论上必定>0，且接近于0，因此接下来可以使 tcp 进入TIME_WAIT状态，时长是1分钟。所以我们只能通过修改内核参数重编译的方式去修改TCP_TIMEWAIT_LEN值，从而影响TIME_WAIT的时间长度。
				tcp_time_wait(sk, TCP_FIN_WAIT2, tmo); // 进入TIME_WAIT状态
				goto out;
			}
		}
		tcp_send_active_reset(sk, GFP_ATOMIC); // 发送RST
		goto death;
	}

	if (!sock_flag(sk, SOCK_KEEPOPEN) ||
	    ((1 << sk->sk_state) & (TCPF_CLOSE | TCPF_SYN_SENT)))
		goto out;

	elapsed = keepalive_time_when(tp); // 获取配置tcp_keepalive_time，默认是2小时

	/* It is alive without keepalive 8) */
	if (tp->packets_out || !tcp_write_queue_empty(sk)) // 还有未发送的数据说明当前的sock是活跃的
		goto resched;

	elapsed = keepalive_time_elapsed(tp); // 到这里说明当前的sock已经是非活跃的，那么此处计算的是已经多久不活跃了

	if (elapsed >= keepalive_time_when(tp)) { // 已经超过了配置时间，那么就要开启心跳检查了
		/* If the TCP_USER_TIMEOUT option is enabled, use that
		 * to determine when to timeout instead.
		 */
		if ((icsk->icsk_user_timeout != 0 &&
		    elapsed >= msecs_to_jiffies(icsk->icsk_user_timeout) &&
		    icsk->icsk_probes_out > 0) ||
		    (icsk->icsk_user_timeout == 0 &&
		    icsk->icsk_probes_out >= keepalive_probes(tp))) { // 如果开启了TCP_USER_TIMEOUT
			tcp_send_active_reset(sk, GFP_ATOMIC); 
			tcp_write_err(sk);
			goto out;
		}
		if (tcp_write_wakeup(sk, LINUX_MIB_TCPKEEPALIVE) <= 0) { // 发送keepalive包成功
			icsk->icsk_probes_out++; // 已发送次数+1
			elapsed = keepalive_intvl_when(tp); // 计算下一次的发送间隔，这里读取配置tcp_keepalive_intvl（默认值是75）
		} else {
			/* If keepalive was lost due to local congestion,
			 * try harder.
			 */
			elapsed = TCP_RESOURCE_PROBE_INTERVAL; // 发送失败，再重试
		}
	} else {
		/* It is tp->rcv_tstamp + keepalive_time_when(tp) */
		elapsed = keepalive_time_when(tp) - elapsed; // 虽然是不活跃的连接，但是不活跃时间还没达到配置的2小时，因此计算差值，等差值时间后再试
	}

resched:
	inet_csk_reset_keepalive_timer (sk, elapsed); // 重设心跳定时器
	goto out;

death:
	tcp_done(sk); // 关闭tcp

out:
	bh_unlock_sock(sk);
	sock_put(sk);
}
```
### tcp_rcv_state_process
当客户端发出`ACK`包后，如果服务端成功接收到了该数据包，那么会执行以下的处理流程
```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
	// ....
	switch (sk->sk_state) {
	case TCP_SYN_RECV:
		tp->delivered++; /* SYN-ACK delivery isn't tracked in tcp_ack */
		if (!tp->srtt_us)
			tcp_synack_rtt_meas(sk, req); // 更新rtt时间

		if (req) {
			tcp_rcv_synrecv_state_fastopen(sk); // 这里虽然名字叫fastopen，实际处理的是request_sock的逻辑，里面会释放掉request_sock
		} else { // 这部分的逻辑暂时不明TODO
			tcp_try_undo_spurious_syn(sk);
			tp->retrans_stamp = 0;
			tcp_init_transfer(sk, BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB,
					  skb);
			WRITE_ONCE(tp->copied_seq, tp->rcv_nxt);
		}
		smp_mb();
		tcp_set_state(sk, TCP_ESTABLISHED); // 服务端的socket状态转换为ESTABLISHED
		sk->sk_state_change(sk);
        // ....
		tp->snd_una = TCP_SKB_CB(skb)->ack_seq; // 更新序列号
		tp->snd_wnd = ntohs(th->window) << tp->rx_opt.snd_wscale; // 发送窗口调整
		tcp_init_wl(tp, TCP_SKB_CB(skb)->seq);
    	// ....
		break;
    }
}
```
### 第三次握手丢失，会发生什么？
由于客户端此时已经接收到了`SYN-ACK`报文，因此成功进入了`ESTABLISHED`状态，然后会向服务端发送一个`ACK`报文，但是注意，**ACK 报文是不会重发的，因此当该报文丢失之后只能等待对方重发相应的报文再促使客户端下发 ACK 报文**。
所以第三次握手包丢失，则会使服务端一直重发`SYN-ACK`报文，直到收到回复或者达到最大重传次数（`tcp_synack_retries（默认值是5）`）。
如果服务端最终因为达到了最大重发次数而转为`CLOSED`状态，那么此时处于`ESTABLISHED`状态的客户端如果没有开启 keepalive 机制（连接空闲2小时之后，开始探测，每次间隔75秒，总共探测9次）者没有进行任何数据包的发送，那么客户端就会一直处于当前状态，直到进程关闭。否则则会由于心跳超时或者数据包重传达到上限而关闭连接，数据包的重传由以下两个参数控制：
```c
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_retries1
3
rxsi@VM-20-9-debian:~$ cat /proc/sys/net/ipv4/tcp_retries2
15
```
在源码`tcp_write_timeout`函数中进行了判断处理：
```c
static int tcp_write_timeout(struct sock *sk)
{
    // ...
	if ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) {
        // ... SYN_SENT、SEN_RECV包的重传处理
	} else {
		if (retransmits_timed_out(sk, READ_ONCE(net->ipv4.sysctl_tcp_retries1), 0)) { // 普通数据包，当该包的重传时间已经大于通过tcp_retries1和rto计算出来的最大重传时间时，需要更新路由缓存，以尝试查找一条更快速的路由路径
			/* Black hole detection */
			tcp_mtu_probing(icsk, sk);

			__dst_negative_advice(sk); // 更新路由
		}

		retry_until = READ_ONCE(net->ipv4.sysctl_tcp_retries2);
        // ....
	}
	if (!expired)
		expired = retransmits_timed_out(sk, retry_until,
						icsk->icsk_user_timeout); // 当该包的重传时间已经大于tcp_retries2和rto计算出的最大重传时间（900多秒），那么则放弃重传，关闭tcp
    // .....
	if (expired) {
		/* Has it gone just too far? */
		tcp_write_err(sk);
		return 1;
	}
    // ....
}
```