---
title: 云场景下的网络QoS
cover: https://s2.loli.net/2022/12/24/FI1YkGehQyVKs9T.jpg
tags: Computer Network
---

公共的网络链路总会不可避免的产生带宽抢占的问题，我们通常使用QoS技术保障大多数用户的服务质量。

![img](https://static001.geekbang.org/resource/image/74/11/747b0d537fd1705171ffcca3faf96211.jpg?wh=1539*646)

一台服务器能控制的只有出方向的QoS，通过Shaping将出站流量整形，至于入栈流量只能通过Policy决定丢弃哪一部分数据包。

## 队列方式控制网络QoS

Linux下的TC就主要是使用队列技术控制的网络QoS。

### 无类别排队规则

不分类（或称无类别）排队规则（classless queueing disciplines）可以对某个网络 接口（interface）上的所有流量进行**无差别整形**。包括对数据进行：

- 重新调度（reschedule）
- 增加延迟（delay）
- 丢弃（drop）

与 classless qdisc 对应的是 classful qdisc，即有类别（或称分类别）排队规则。目前最常用的classless qdisc 是**pfifo_fast**，这也是很多系统上的 默认排队规则。

##### pfifo_fast 先入先出队列

如名字所示，这是一个先入先出(FIFO)队列，因此对所有的包都一视同仁。

![img](https://arthurchiao.art/assets/img/lartc-qdisc/pfifo_fast-qdisc.png)

pfifo_fast有三个所谓的 “band”（可理解为三个队列），编号分别为 0、1、2：

![img](https://static001.geekbang.org/resource/image/e3/6c/e391b4b79580a7d66afe4307ff3f6f6c.jpg?wh=2037*1175)

- 每个band上分别执行 FIFO 规则。
- 如果band 0有数据，就不会处理band 1；同理，band 1有数据时，不会去处理band 2。
- 内核会检查数据包的 `TOS`字段，将“最小延迟”的包放到band 0。



##### Stochastic Fair Queuing (随机公平队列)

![img](https://static001.geekbang.org/resource/image/b6/99/b6ec2e4e20ddee7d6952b7fa4586ba99.jpg?wh=2177*1182)

这种无类别队列规则会建立很多的 FIFO 的队列，TCP Session 会计算 hash 值，通过 hash 值分配到某个队列。在队列的另一端，网络包会通过轮询策略从各个队列中取出发送。这样不会有一个 Session 占据所有的流量。

当然，如果两个 Session 的 hash 是一样的，会共享一个队列，也有可能互相影响。hash 函数会经常改变，从而 session 不会总是相互影响。



##### Tolen Bucket Filte (TBF,令牌桶规则)

![img](https://static001.geekbang.org/resource/image/14/9b/145c6f8593bf7603eae79246b9d6859b.jpg?wh=1894*1100)

所有的网络包排成队列进行发送，但不是到了队头就能发送，而是需要拿到令牌才能发送。

令牌根据设定的速度生成，所以即便队列很长，也是按照一定的速度进行发送的。当没有包在队列中的时候，令牌还是以既定的速度生成，但是不是无限累积的，而是放满了桶为止。

设置桶的大小为了避免下面的情况：当长时间没有网络包发送的时候，积累了大量的令牌，突然来了大量的网络包，每个都能得到令牌，造成瞬间流量大增。