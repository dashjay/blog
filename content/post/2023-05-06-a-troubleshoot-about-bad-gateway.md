---
title: "定位 502 问题，最后竟然发现和交换机有关"
date: 2023-05-06T11:27:44+08:00
draft: false
slug: troubleshoot-502-problem 
---

## 0x00 起因

昨天有人报告说使用我们的服务间歇性抽风出现了一些 `502` 报错，查了一下 `kibana` 发现日志里确实出现了一些，上服务器检查了一下发现有如下日志：

> 为了脱敏，我把一些 IP 细节隐藏掉了，本文涉及的服务一共有四个节点，分别是`proxy`服务，下面替换为`ip-proxy`；和三个`backend`，但是他们位于一个交换机下，并且交换机给他们配置了一个`VIP`，我们称为，`ip-backend`。

```
May 05 17:33:07 *: 2023/05/05 17:33:07 http: proxy error: read tcp ip-proxy:51392->ip-backend:80: read: connection reset by peer
May 05 17:38:09 *: 2023/05/05 17:38:09 http: proxy error: read tcp ip-proxy:52468->ip-backend:80: read: connection reset by peer
May 05 17:39:14 *: 2023/05/05 17:39:14 http: proxy error: readfrom tcp ip-proxy:52736->ip-backend:80: write tcp ip-proxy:52736->ip-backend:80: use of closed network connection
....
18.32.11:80: use of closed network connection
May 05 15:03:34 *: 2023/05/05 15:03:34 http: proxy error: readfrom tcp ip-proxy:56344->ip-backend:80: write tcp ip-proxy:56344->ip-backend:80: write: broken pipe
May 05 15:03:35 *: 2023/05/05 15:03:35 http: proxy error: readfrom tcp ip-proxy:56364->ip-backend:80: write tcp ip-proxy:56364->ip-backend:80: write: broken pipe
May 05 15:03:36 *: 2023/05/05 15:03:36 http: proxy error: readfrom tcp ip-proxy:56388->ip-backend:80: write tcp ip-proxy:56388->ip-backend:80: write: connection reset by peer
May 05 15:03:38 *: 2023/05/05 15:03:38 http: proxy error: readfrom tcp ip-proxy:56462->ip-backend:80: write tcp ip-proxy:56462->ip-backend:80: use of closed network connection
May 05 15:03:40 *: 2023/05/05 15:03:40 http: proxy error: readfrom tcp ip-proxy:56534->ip-backend:80: write tcp ip-proxy:56534->ip-backend:80: use of closed network connection
May 05 15:03:47 *: 2023/05/05 15:03:47 http: proxy error: readfrom tcp ip-proxy:56796->ip-backend:80: write tcp ip-proxy:56796->ip-backend:80: use of closed network connection
May 05 15:03:48 *: 2023/05/05 15:03:48 http: proxy error: readfrom tcp ip-proxy:56860->ip-backend:80: write tcp ip-proxy:56860->ip-backend:80: write: connection reset by peer
May 05 15:03:50 *: 2023/05/05 15:03:50 http: proxy error: readfrom tcp ip-proxy:56870->ip-backend:80: write tcp ip-proxy:56870->ip-backend:80: write: connection reset by peer
May 05 15:03:50 *: 2023/05/05 15:03:50 http: proxy error: readfrom tcp ip-proxy:56908->ip-backend:80: write tcp ip-proxy:56908->ip-backend:80: use of closed network connection
```

说实话，看到这个报错，我第一时间去看了我们代码，全文搜索过后，我并没有在任何地方返回 502。

第一时间根据日志推测出来大致可能性： `proxy` 到3个 `backend` 的连接异常断开了，然后 `proxy` 中使用了 httputil.ReverseProxy 相关代码，内部有类似 Nginx 反向代理的逻辑，导致了这个问题，我开始在代码里翻找。

## 0x01 查代码——冤假错案

我仔细观察代码，找到可能产生错误 close 连接的代码，并没有找到，然后突然我看到了这样的代码：

```go
func newProxyTransport() *http.Transport {
	return &http.Transport{
		DialContext: (&net.Dialer{
			Timeout:   10 * time.Second,
			KeepAlive: 100 * time.Second,
		}).DialContext,
		MaxIdleConns:          4096,
		MaxIdleConnsPerHost:   4096,
		IdleConnTimeout:       90 * time.Second,
		TLSHandshakeTimeout:   10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
	}
}
```

我发现 `KeepAlive` 比 `IdleConnTimeout` 时间长这并不符合逻辑基本逻辑，我特地看了 `Golang` 的 `doc`。

KeepAlive：指定了内部两次网络探活检测的间隔。
```
	// KeepAlive specifies the interval between keep-alive
	// probes for an active network connection.
	// If zero, keep-alive probes are sent with a default value
	// (currently 15 seconds), if supported by the protocol and operating
	// system. Network protocols or operating systems that do
	// not support keep-alives ignore this field.
	// If negative, keep-alive probes are disabled.
```

IdleConnTimeout：使用后在关闭前最大的空闲时间。
```
	// IdleConnTimeout is the maximum amount of time an idle
	// (keep-alive) connection will remain idle before closing
	// itself.
	// Zero means no limit.
```

我开始假想出现这样的问题：假设连接空闲了 `90` 秒，要被系统关闭，但是没有等到一次 `Keep-Alive` 检测，服务刚好要使用这个连接，连接就刚好关闭了，应该是偶发现象，心安理得，开始摸鱼。

随后第二天来我发现事情越来越不可控了，`502` 的问题越来越多，我不得不继续排查，不得不选择最最最不想做的抓包……再额外交代一下背景就是这个破玩意儿流量非常大，几分钟就能抓几十个G的包。

### 0x02 抓包——当面对峙

抓包是一个极其无聊的过程……，而且因为业务流量大、偶发（一个小时一次或几次）的问题，我不得不像守株待兔一样，盯着这破东西，我也想过用 `gopackage` 或者 `eBPF` 来写一些基础的 filter，最终还是决定就硬抓算了。

每隔几分钟就开新的，并且关闭旧的抓包，这样 rotate 抓包，抓了不知道多久，日志里突然跳出来一个报错，`use of closed network`，我立马关闭抓包，可是还是抓了总计100多G的包……

在三个 backend 节点上分别抓到30多G的包，管他的直接开撸，了解到报错的端口号之后：

```
tshark -r 001.cap -Y 'ip.addr==ip-proxy and tcp.port==49092'


3841744 220.895866 ip-proxy → ip-backend HTTP 3501 POST /xxxxxxxxxxxxxxxxxxxxxxxxx......
3841745 220.895866 ip-proxy → ip-backend TCP 3501 [TCP Retransmission] 49092 → 80 [PSH, ACK] Seq=4161 Ack=1 Win=42496 Len=3433 TSval=1825820048 TSecr=2612233213
3841746 220.895879 ip-backend → ip-proxy TCP 68 80 → 49092 [ACK] Seq=1 Ack=1552 Win=34816 Len=0 TSval=2612233213 TSecr=1825820048
3841747 220.895880 ip-backend → ip-proxy TCP 68 [TCP Dup ACK 3841746#1] 80 → 49092 [ACK] Seq=1 Ack=1552 Win=34816 Len=0 TSval=2612233213 TSecr=1825820048
3841748 220.895904 ip-backend → ip-proxy TCP 68 80 → 49092 [ACK] Seq=1 Ack=4161 Win=34816 Len=0 TSval=2612233213 TSecr=1825820048
3841749 220.895905 ip-backend → ip-proxy TCP 68 [TCP Dup ACK 3841748#1] 80 → 49092 [ACK] Seq=1 Ack=4161 Win=34816 Len=0 TSval=2612233213 TSecr=1825820048
3841750 220.895937 ip-backend → ip-proxy TCP 68 80 → 49092 [ACK] Seq=1 Ack=7594 Win=34816 Len=0 TSval=2612233214 TSecr=1825820048
3841751 220.895938 ip-backend → ip-proxy TCP 68 [TCP Dup ACK 3841750#1] 80 → 49092 [ACK] Seq=1 Ack=7594 Win=34816 Len=0 TSval=2612233214 TSecr=1825820048
3841803 220.901916 ip-backend → ip-proxy HTTP 1182 HTTP/1.1 200 OK  (text/plain)
3841805 220.901918 ip-backend → ip-proxy TCP 1182 [TCP Retransmission] 80 → 49092 [PSH, ACK] Seq=1 Ack=7594 Win=34816 Len=1114 TSval=2612233219 TSecr=1825820048
3841812 220.901996 ip-proxy → ip-backend TCP 68 49092 → 80 [ACK] Seq=7594 Ack=1115 Win=42496 Len=0 TSval=1825820054 TSecr=2612233219
3841813 220.901996 ip-proxy → ip-backend TCP 68 [TCP Dup ACK 3841812#1] 49092 → 80 [ACK] Seq=7594 Ack=1115 Win=42496 Len=0 TSval=1825820054 TSecr=2612233219
3841962 220.904543 ip-proxy → ip-backend TCP 1516 POST /xxxxxxxxxxxxxxxxx HTTP/1.1  [TCP segment of a reassembled PDU]
3841963 220.904543 ip-proxy → ip-backend TCP 1516 [TCP Retransmission] 49092 → 80 [ACK] Seq=7594 Ack=1115 Win=42496 Len=1448 TSval=1825820057 TSecr=2612233219
3841964 220.904630 ip-backend → ip-proxy TCP 68 80 → 49092 [ACK] Seq=1115 Ack=9042 Win=34816 Len=0 TSval=2612233222 TSecr=1825820057
3841965 220.904631 ip-backend → ip-proxy TCP 68 [TCP Dup ACK 3841964#1] 80 → 49092 [ACK] Seq=1115 Ack=9042 Win=34816 Len=0 TSval=2612233222 TSecr=1825820057
3841966 220.904666 ip-proxy → ip-backend TCP 62 49092 → 80 [RST] Seq=9042 Win=0 Len=0
3841967 220.904666 ip-proxy → ip-backend TCP 62 49092 → 80 [RST] Seq=9042 Win=0 Len=0
```

通过 3841962，和63 这2行可以看到：`ip-proxy` 帮忙向 `ip-backend` 转发了一个 POST 请求，并且重传了一次，然后 `ip-backend` 向 `ip-proxy` 发送了两个 `ACK`，然后客户端的就 `RST` 了？？？？

这是为什么？请求还没返回，百思不得其解！

突然想到一个之前看过的文章

文章在这里：[动图图解！收到RST，就一定会断开TCP连接吗？—— 作者：小白debug](https://zhuanlan.zhihu.com/p/422383933)

文章传达了类似：RST 可以是攻击者伪造的这样的观点，我就在想，TCP 这种面向连接的传输协议没被挟持的情况下还能被第三方攻击？而且文章里面提到了出现RST的原因：“RST一般出现于异常情况，归类为 **对端的端口不可用** 和 **socket提前关闭**。”

根据目前的情况来看，`ip-backend` 的服务端口是可用的，这个 `ip-proxy` 和 `ip-backend` 之间看起来似乎是，有一方先关闭了，从抓包来看，应该是 `ip-proxy` 这一方关闭了，导致 `ip-backend` 收到对方的 `ACK` 之后直接 RST 了。

目前暂无头绪，画个图分析一下，大佬们博客中的说法。

```
┌───────────┐             ┌───────────┐            ┌───────────┐
│           │             │           │            │           │
│   Cliet   │             │   Proxy   │            │  Backend  │
│           │             │           │            │           │
└─────┬─────┘             └─────┬─────┘            └─────┬─────┘
      │        Request          │       Request          │
      ├───────────────────────► ├───────────────────────►│
      │                         │                        │
      │        Response         │       Response         │
      │◄────────────────────────┤◄───────────────────────┤
      │                         │                        │
      │                         │                        │
      │x x x x x x x x x x x x x│x x x x x x x x x x x x │
      │        Request          │       Request          │
      ├───────────────────────► ├───────────────────────►│
      │                         │                        │
      │        502              │       RST              │
      │◄────────────────────────┤◄───────────────────────┤
      │                         │                        │
      │                         │                        │
      │                         │                        │
      │                         │                        │
```

可是为什么好端端的 Backend 要返回 RST 呢？

又一次陷入了停滞……

不过到目前为止我的认识还在应用层和传输层，从这两层来看，TCP 包被客户端 RST 了是百思不得其解的，我们可以大致认为有两种可能：

1. `Proxy` 的系统出现了一些问题，导致 `Backend` 发给 `Proxy` 的包被 `Proxy` 主动 RST 了
2. `Backned` 事先出了一些问题，给 `Proxy` 发了 `RST`，导致 `Proxy` 这边的连接被单方面断开

如果是 1 的话，那么Proxy 这边的系统肯定有 Bug，频发连接失效导致直接发 RST（我通常是不会怀疑到操作系统上的）；

那么只可能是 2，如果是 2 的话，Backend 向 Proxy 发的 RST 包应该被抓到呀，想到这里，我突然想到了一种可能性，我执行了如下命令：

```
tshark -r 002.cap -Y 'ip.addr==ip-proxy and tcp.port==49092'
tshark -r 003.cap -Y 'ip.addr==ip-proxy and tcp.port==49092'
3113698 215.885959 ip-proxy → ip-backend TCP 172 49092 → 80 [PSH, ACK] Seq=1 Ack=1 Win=83 Len=104 TSval=1825820057 TSecr=2612233219
3113699 215.885959 ip-proxy → ip-backend TCP 172 [TCP Retransmission] 49092 → 80 [PSH, ACK] Seq=1 Ack=1 Win=83 Len=104 TSval=1825820057 TSecr=2612233219
3113700 215.885984 ip-backend → ip-proxy TCP 56 80 → 49092 [RST] Seq=1 Win=0 Len=0
3113701 215.885985 ip-backend → ip-proxy TCP 56 80 → 49092 [RST] Seq=1 Win=0 Len=0
```

在查询了2、3两个 Backend 两个节点上抓到的包，果然发现了端倪。


我们仔细看三组抓包会发现：

```
tshark -r 001.cap -Y 'ip.addr==ip-proxy and tcp.port==49092'
.....
3841962 220.904543 ip-proxy → ip-backend TCP 1516 POST /xxxxxxxxxxxxxxxxx HTTP/1.1  [TCP segment of a reassembled PDU]
3841963 220.904543 ip-proxy → ip-backend TCP 1516 [TCP Retransmission] 49092 → 80 [ACK] Seq=7594 Ack=1115 Win=42496 Len=1448 TSval=1825820057 TSecr=2612233219
.....
tshark -r 003.cap -Y 'ip.addr==ip-proxy and tcp.port==49092'
3113698 215.885959 ip-proxy → ip-backend TCP 172 49092 → 80 [PSH, ACK] Seq=1 Ack=1 Win=83 Len=104 TSval=1825820057 TSecr=2612233219
3113699 215.885959 ip-proxy → ip-backend TCP 172 [TCP Retransmission] 49092 → 80 [PSH, ACK] Seq=1 Ack=1 Win=83 Len=104 TSval=1825820057 TSecr=2612233219
3113700 215.885984 ip-backend → ip-proxy TCP 56 80 → 49092 [RST] Seq=1 Win=0 Len=0
3113701 215.885985 ip-backend → ip-proxy TCP 56 80 → 49092 [RST] Seq=1 Win=0 Len=0
```

这里的 001.cap 和 003.cap 是在不同节点上抓到的，虽然它的 ip 都是 ip-backend，但是其实是指向不同节点的。

仔细看上面的包会发现，ip-proxy 向ip-backend-01 号节点发了一个 POST 请求，长度为 1516，然后可能因为 MTU 是1500的原因？这个包被分成了两个部分，第一部分被正常的发给了 ip-backend-01 节点，然后紧接着第二段发到了 ip-backend-03 号节点上，该节点意识到这不是一个正常的已建立的连接之后，给返回了 RST，告诉对端你可能搞错了。然后紧接着 ip-proxy 就把该链接关闭了，结果刚好这时候 ip-backend-01 节点返回的 ACK 就被当做异常报文给 RST 了，整个转发过程就 502 了。

咨询了一下运维，听说我们的交换机是 `Mellanox` 的软件交换机，对此并不了解，但是目前可以断定，应该是交换机等基础设施出了问题。
