---
layout: default
title:  "Linux Syn SynAck重传机制简析"
categories: Linux Tcp
---

### Linux协议栈 Syn SynAck重传机制简析



以下分析基于3.10版本的内核

#### Syn重传

首先来看Syn包的重传，Syn包的发送是由应用层通过connect调用来触发的，最终进入到`tcp_v4_connect`这个函数，然后调用`tcp_connect`这个函数，该函数如下：

```c
/* Build a SYN and send it off. */
int tcp_connect(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *buff;
    int err;

    tcp_connect_init(sk);

    if (unlikely(tp->repair)) {
        tcp_finish_connect(sk, NULL);
        return 0;
    }

    buff = sk_stream_alloc_skb(sk, 0, sk->sk_allocation, true);
    if (unlikely(!buff))
        return -ENOBUFS;

    tcp_init_nondata_skb(buff, tp->write_seq++, TCPHDR_SYN);
    tp->retrans_stamp = tcp_time_stamp;
    tcp_connect_queue_skb(sk, buff);
    tcp_ecn_send_syn(sk, buff);

    /* Send off SYN; include data in Fast Open. */
    err = tp->fastopen_req ? tcp_send_syn_data(sk, buff) :
          tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);
    if (err == -ECONNREFUSED)
        return err;

    /* We change tp->snd_nxt after the tcp_transmit_skb() call
     * in order to make this packet get counted in tcpOutSegs.
     */
    tp->snd_nxt = tp->write_seq;
    tp->pushed_seq = tp->write_seq;
    TCP_INC_STATS(sock_net(sk), TCP_MIB_ACTIVEOPENS);

    /* Timer for repeating the SYN until an answer. */
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                  inet_csk(sk)->icsk_rto, TCP_RTO_MAX);
    return 0;
}
```

构造了一个syn包，然后加入到sock的发送队列，启动重传定时器，所以从这里我们可以知道**syn包的重传是重传定时器在处理**。

在tcp重传定时器处理函数`tcp_retransmit_timer`中，对syn包的处理有如下特点：

- 首次定时器超时的时间是`TCP_TIMEOUT_INIT`
- 后续超时后超时时间指数回退，直到**重传次数**超过了`sysctl_tcp_syn_retries`，重传结束，关闭sock



另外，顺便在想在这里讨论一下linux下Tcp connect的行为，应用层如果调用了阻塞的connect，在一直没有建连成功的情况下，connect多久会返回？

通过上面代码的分析，我们知道syn重传的时间是和`sysctl_tcp_syn_retries`相关的，也就是在用完重传次数后就会返回。但也可以通过setsockopt设置`SO_SNDTIMEO`这个选项来设置建连的超时时间，更常见的做法是在应用层通过调用非阻塞的connect，然后通过select来控制建连时间。



#### SynAck重传

首先看一下内核是怎样发出synack的，在收到syn包后，最终进入`tcp_v4_conn_request`函数，然后将新建的request_sock加入到监听sock的SYN table中，随后，会在keepalive timer中进行对synack的重传，代码如下：

```c
    if (likely(!do_fastopen)) {
        int err;
        err = ip_build_and_send_pkt(skb_synack, sk, ireq->loc_addr,
             ireq->rmt_addr, ireq->opt);
        err = net_xmit_eval(err);
        if (err || want_cookie)
            goto drop_and_free;

        tcp_rsk(req)->snt_synack = tcp_time_stamp;
        tcp_rsk(req)->listener = NULL;
        /* Add the request_sock to the SYN table */
        inet_csk_reqsk_queue_hash_add(sk, req, TCP_TIMEOUT_INIT);
        if (fastopen_cookie_present(&foc) && foc.len != 0)
            NET_INC_STATS_BH(sock_net(sk),
                LINUX_MIB_TCPFASTOPENPASSIVEFAIL);
    } else if (tcp_v4_conn_req_fastopen(sk, skb, skb_synack, req))
        goto drop_and_free;

    return 0;
```



如上在发出synack后，`inet_csk_reqsk_queue_hash_add`将req sock加入到SYN table，同时给的初始超时时间为`TCP_TIMEOUT_INIT`

```c
void inet_csk_reqsk_queue_hash_add(struct sock *sk, struct request_sock *req,
                   unsigned long timeout)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct listen_sock *lopt = icsk->icsk_accept_queue.listen_opt;
    const u32 h = inet_synq_hash(inet_rsk(req)->rmt_addr, inet_rsk(req)->rmt_port,
                     lopt->hash_rnd, lopt->nr_table_entries);

    reqsk_queue_hash_req(&icsk->icsk_accept_queue, h, req, timeout);
    inet_csk_reqsk_queue_added(sk, timeout);
}
```

随后会在keepalive timer中处理syn ack的重传

最终是在keepalive timer中的`inet_csk_reqsk_queue_prune`函数中处理

```c
static void tcp_synack_timer(struct sock *sk)
{
    inet_csk_reqsk_queue_prune(sk, TCP_SYNQ_INTERVAL,
                   TCP_TIMEOUT_INIT, TCP_RTO_MAX);
}
```

该函数给了三个时间相关的参数，第一个`TCP_SYNQ_INTERVAL`是keepalive timer的执行间隔，`TCP_TIMEOUT_INIT`是synack重传的基础时间，会随着重传次数在这基础上进行指数回退，`TCP_RTO_MAX`是重传最大的时间间隔。

核心部分代码如下：

```c
do {
        reqp=&lopt->syn_table[i];
        while ((req = *reqp) != NULL) {
            if (time_after_eq(now, req->expires)) {
                int expire = 0, resend = 0;

                syn_ack_recalc(req, thresh, max_retries,
                           queue->rskq_defer_accept,
                           &expire, &resend);
                req->rsk_ops->syn_ack_timeout(parent, req);
                if (!expire &&
                    (!resend ||
                     !inet_rtx_syn_ack(parent, req) ||
                     inet_rsk(req)->acked)) {
                    unsigned long timeo;

                    if (req->num_timeout++ == 0)
                        lopt->qlen_young--;
                    timeo = min(timeout << req->num_timeout,
                            max_rto);
                    req->expires = now + timeo;
                    reqp = &req->dl_next;
                    continue;
                }

                /* Drop this request */
                inet_csk_reqsk_queue_unlink(parent, req, reqp);
                reqsk_queue_removed(queue, req);
                reqsk_free(req);
                continue;
            }
            reqp = &req->dl_next;
        }

        i = (i + 1) & (lopt->nr_table_entries - 1);

    } while (--budget > 0);
```

这里有个问题需要注意一下，这里也包含了对defer accept的处理，看函数`syn_ack_recalc`：

```c
/* Decide when to expire the request and when to resend SYN-ACK */
static inline void syn_ack_recalc(struct request_sock *req, const int thresh,
                  const int max_retries,
                  const u8 rskq_defer_accept,
                  int *expire, int *resend)
{
    if (!rskq_defer_accept) {
        *expire = req->num_timeout >= thresh;
        *resend = 1;
        return;
    }
    *expire = req->num_timeout >= thresh &&
          (!inet_rsk(req)->acked || req->num_timeout >= max_retries);
    /*
     * Do not resend while waiting for data after ACK,
     * start to resend on end of deferring period to give
     * last chance for data or ACK to create established socket.
     */
    *resend = !inet_rsk(req)->acked ||
          req->num_timeout >= rskq_defer_accept - 1;
}
```

在没有设置defer accept的情况下，很简单，是否超时就看重传次数是否超过了限制（由sysctl_tcp_synack_retries决定），重传一直为1.

在设置了defer accept的情况下，首先看是否超时不但要满足重传次数超过了sysctl_tcp_synack_retries的限制，而且在收到ack的情况下（defer accept开始起作用），重传次数还要超过另外一个`max_retries`的限制，是否重传也在收到ack情况下与`rskq_defer_accept`有关，这个`rskq_defer_accept`在defer accept开启的情况下是和配置的defer accept的时间相关的，代码如下，在setsockopt时进行设置的：

```c
    case TCP_DEFER_ACCEPT:
        /* Translate value in seconds to number of retransmits */
        icsk->icsk_accept_queue.rskq_defer_accept =
            secs_to_retrans(val, TCP_TIMEOUT_INIT / HZ,
                    TCP_RTO_MAX / HZ);
        break;

```



```c
/* Convert seconds to retransmits based on initial and max timeout */
static u8 secs_to_retrans(int seconds, int timeout, int rto_max)
{
    u8 res = 0;

    if (seconds > 0) {
        int period = timeout;

        res = 1;
        while (seconds > period && res < 255) {
            res++;
            timeout <<= 1;
            if (timeout > rto_max)
                timeout = rto_max;
            period += timeout;
        }
    }
    return res;
}
```

计算次数的方式是从上层传递的时间换算为次数，换算的方式基本为按照指数回退的方式一直回退到rto_max，计算给定的时间能够重传的次数。

