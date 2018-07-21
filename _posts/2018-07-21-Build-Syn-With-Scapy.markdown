---
layout: default
title:  "使用Python Scapy模拟Tcp三次握手"
categories: Scapy Tcp
---

### 使用Python Scapy模拟Tcp三次握手



Python的Scapy是一个非常强大的抓包及构造包的库，这片文章介绍如何使用Scapy构造Tcp报文来模拟三次握手，以及Tcp Syn包中一些Option如何设置。



构建Syn包代码如下：

```python
from scapy.all import *

# SYN
ip = IP(src = src, dst = dst)

syn = TCP(sport=sport, dport=dport, flags="S", seq=38938, options=[('MSS', 1460), ('NOP', None), ('NOP', None), ('SAckOK', ''), ('NOP', None), ('WScale', 4)])

synack = sr1(ip/syn)
```

其中options包含了对MSS的设置，对Sack的支持，对WScale的调整。

然后通过sr1收取到了对端返回的synack，然后通过synack的信息返回ack的代码如下：

```python
ack = TCP(sport = sport, dport = dport, flags = 'A', seq = synack.ack, ack = synack.seq + 1)
send(ip/ack/payload)
```

当然，我们也可以在返回的ack中携带请求数据，代码如下：

```python
payload=b'GET / HTTP/1.1\r\nUser-Agent: Wget/1.14 (linux-gnu)\r\nAccept: */*\r\nHost: 1.1.1.1\r\nConnection: Keep-Alive\r\n\r\n'

ack = TCP(sport = sport, dport = dport, flags = 'A', seq = synack.ack + len(payload), ack = synack.seq + 1 + 13350)
send(ip/ack)
```



