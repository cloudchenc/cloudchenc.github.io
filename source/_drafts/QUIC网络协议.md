---
title: QUIC网络协议
tags:
    - TCP
    - UDP
categories:
---

RTT(Round Trip Time)：一个连接的往返时间，即数据发送时刻到接收到确认的时刻的差值；
RTO(Retransmission Time Out)：重传超时时间，即从数据发送时刻算起，超过这个时间便执行重传。

一次HTTPS请求

TCP三次握手，消耗1.5RTT
SSL/TLS 四次通信，消耗 2RTT
HTTP request/response，消耗 1RTT

共消耗4.5RTT

一次QUIC请求 Technology

建立连接，消耗 1RTT
QUIC request/resonse，消耗 1RTT

QUIC 非常类似于在 UDP 上实现的 TCP + TLS + HTTP/2。

QUIC实现了可靠且稳定的UDP连接。

性能：QUIC在总体性能上比HTTP2略有提升，在弱网络、高丢包率的情况下，QUIC才能凸显其独特优势。

HTTP2痛点：
1. 队头阻塞
2. TCP 和 TLS 握手时延
3. 连接迁移需要重新连接（四元组结构，双方ip及端口）

QUIC解决了：
1. 无队头阻塞
2. 建立连接速度快
3. 连接迁移

应用：

1. HTTP3，弱网环境下也可以流畅访问了，未来3-5年内可能会普及
2. 低延时直播，RTMP over QUIC，延时从2s降低到800ms
3. 即时通信（QQ、WeChat目前类似email，存在一定的延迟，还不是真正意义上的通信），网络游戏以及物联网（远程设备网络控制及通信）等都会变得有可能

https://cloud.tencent.com/developer/article/1405624

腾讯落地实践：https://cloud.tencent.com/developer/article/1198399
蚂蚁落地实践：https://cloud.tencent.com/developer/article/1822608

QUIC实际落地中，会碰到哪些问题？

1. 为了支持连接迁移，设计了ServiceInfo结构体，提供路由到的服务器信息，包括ip和port
2. token一致问题，使用集群共享校验，另外设计client id和client ip的概念，进行双重验证
3. UDP无损升级？？？
4. 客户端智能选路，实时监测TCP链路和QUIC链路，动态选择

