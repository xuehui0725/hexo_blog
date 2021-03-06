---
title: 分布式系统时序基础
date: 2016-07-02 10:38:25
categories: 分布式
tags:
  - 分布式
  - 时序
---

# Introduction

本文是学习Time, Clocks, and the Ordering of Events in a Distributed System论文的总结，本文总结了提出了通过total ordering events来解决分布式系统的同步问题。

## What is a distributed system

- 空间分离的进程合集，进程之间只能通过消息交换来通信
- 进程之间的消息通信耗费的时间要远远高于单个进程间的通信时间

# The Partial Ordering

在讨论Partial Ordering之前，先复习下离散数学中关系的特性，其中二元关系<>是针对集合A的。

- 如果对于任意的a属于A，满足a<>a，则二元关系是自反的
- 如果对于任意的a，b属于A，如果满足a<>b，且同时也满足b<>a，则二元关系是对称的
- 如果对于任意的a，b，c属于A，如果满足a<>b且b<>c，也同时满足a<>c，则二元关系是可传递的
- 如果对于任意的a,b属于A，如果满足a<>b且b<>a，则a=b,则二元关系是反对称的

一个partitial ordering关系满足的条件是自反的，对称的和可传递的，因此在partitial ordering中，可能有两个元素之间是不相关的。

一个total ordering关系满足的条件是自反的，对称的，可传递的和反对称的，因此在total ordering中，两个元素一定是有关系的，要么是a<>b或b<>a。

在分布式系统中，一个进程包含一系列的事件，对于同一进程内的事件，如果a happens before b，那么a发生在b之前。并且，假定收或发消息都是一个事件。

happens before的定义如下(用->表示)

1. 如果a和b在同一进程中，并且a发生在b之前，那么a->b
2. 如果a是一个进程发消息的事件，b是另一个进程接收这条消息的事件，则a->b
3. 如果a->b且b->c，那么a->c。
4. 如果同时不满足a->b，且b->a，那么说a和b是concurrent

且假定对于任意的a，都不满足a->a，因为一个事件不可能发生在本身之前。因此可以看出happens before是不满足自反性的partial ordering。

![](http://o8m1nd933.bkt.clouddn.com/blog/time-clock/happens_before.png)

以一个例子来说明happens before关系，如上图，垂直线上代表一个进程，从下往上，时间依次增加，水平的距离代表空间的隔离。原点代表一个事件，而曲线代表一条消息。

从图中很容易地看出，如果一个事件a，能通过进程的线(从下往上)和消息线(从左往右)，到达b，那么a->b，例如，p1->r4。直观地角度看，如果事件a能最终导致事件b发生，则可以说明a->b。

在图中，p3和q4是并行的事件，因为，只有到了p4才能确定q4的发生，而q3也只能确定p1发生。

# Logical Clocks

对于逻辑时钟，对于每个事件会分配一个数字，认为这个数字就是事件发生的时间。

时钟的定义如下

- 对于一个进程i，Ci(a)表示进程i中事件a的发生时间
- 对于整个系统来讲，对于任意的事件b，其发生时间为C(b)，当b为进程j的事件时，则C(b) = Cj(b)

为了使得事件按照正确的排序，需要使得如果事件a发生在事件b之前，那么a发生的时间要小于b，如下

```
for any events a, b
if a->b then C(a) < C(b)
```
根据关系->的定义，我们可以得出

- 如果a和b都是进程i中的事件，且a发生在b之前，那么Ci(a) < Ci(b)
- 如果事件a发送消息给事件b，a属于进程i，b属于进程j，那么Ci(a) < Cj(b)

![](http://o8m1nd933.bkt.clouddn.com/blog/time-clock/logical_time.png)

在上图中，水平虚线代表一次时钟的tick。为了满足上述第一个条件，那么同一个进程中的事件之间必须要有tick线隔开；为了满足第二个条件，那么消息线必须跨过tick线。

为了让系统满足上述条件，在实现中，需要满足以下原则

- 对于每个进程，相邻的事件的时钟要增加1
- (a) 如果事件a是进程i发送消息m的事件，发送时带时间戳Tm = Ci(a)，(b)事件b是进程j接受消息m的事件，那么事件b的取值为max(进程b的当前时钟，Tm+1)

# Ordering the Events Totally

total order的事件关系=>定义如下：

如果事件a发生在进程Pi，事件b发生在进程Pj，那么当满足下列两者条件之一时，a=>b

- Ci(a) < Cj(b)
- Ci(a) = Cj(b) 且 Pi < Pj

根据以上条件，对于任意的两个事件，都能判断出它们之间的关系，因此是total ordering的。

通过一个例子来说明total ordering events的用法。问题为mutual exclusion problem，即一个由一组进程组成的系统，共享一个资源，该资源每次只能让一个进程使用，因此，需要进程间需要作同步防止冲突。需要找一个算法，满足以下条件：

- 某进程被分配了资源后，必须要等该进程释放该资源后，才能再把资源分配给其他进程
- 不同的资源请求会按照它们申请的顺序进行分配
- 如果每个被分配资源的进程最终都会释放，那么每个请求最终都会被分配资源

在设计算法前，假定系统满足如下条件：

- 对于任意两个进程Pi和Pj，从Pi和Pj发送的消息，接受的顺序与发送的顺序一致
- 每个消息最终都会被接收
- 每个进程可以直接发送消息给其他进程

每个进程都维护自己的队列，并且其他进程无法访问。我们假定一开始队列里面包含请求T0:P0，其中P0是最初被分配资源的进程，T0比任意的时钟的初始值都要小。

该算法的5个原则如下

1. 为了请求资源，进程Pi发送消息Tm:Pi给所有的其他进程，并且把这个消息放到进程队列中，Tm是消息的时间戳
2. 当进程Pj接收到了进程Pi的Tm：Pi请求后，会把它放到自己的请求队列，然后发送一个带时间戳的确认消息给Pi
3. 为了释放资源，进程Pi移除所有Tm:Pi的请求消息，然后发送带时间戳的Pi释放资源请求消息给其他所有的进程
4. 当进程Pj接收到进程Pi释放资源的请求，它会移除队列中任意的Tm:Pi的资源请求
5. 当满足以下两个条件时，进程Pi会被分配该资源：a)有一个Tm:Pi的请求，按照=>关系排在队列第一位；b)Pi接收到了一个时间戳大于Tm的来自所有其他进程的消息

# Anomalous behavior

上面章节描述的=>关系可能导致Anomalous behavior，以一个例子来说明。

假如一个由互联的电脑组成的系统，其中有共享的资源。一个用户在电脑A上发送请求共享资源，然后他打电话给朋友然他在电脑B上发送请求B也来请求那个资源。

在电脑B上，如果请求a还未到达之前，已经产生了请求b，那么，在电脑B上，b排在a前面

在电脑A上，则是请求a排在请求b前面，这样就会造成冲突。

有两种方案解决上述冲突：

- 让用户来保证不发生该冲突。例如，当请求a生成后，用户需要分配时间戳给请求a，当用户让他朋友发送请求b的时候，需要保证分配一个大于a的时间戳
- 假设S是系统中所有事件的组成，而S‘是包含S且外部事件的组合。定义==>在集合S’上的happen before，则满足以下条件

```
for any events a,b in S : if a==>b then C(a) < C(b)
```

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://o8m1nd933.bkt.clouddn.com/blog/qcode_wechat.jpg)

# References

- Time, Clocks, and the Ordering of Events in a Distributed System
- [Summary](http://www.ics.uci.edu/~cs230/reading/time.pdf)
