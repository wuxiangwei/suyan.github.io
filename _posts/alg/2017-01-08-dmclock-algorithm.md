---
layout: post
title: 聊聊dmClock算法
category: [算法, 存储]
tags: [dmClock]
keywords: dmClock
---


人们常常容易忽略一些不起眼但特别重要的事物。曾经跟同事聊Python，有人说一切皆对象，这也是一些OOP的广告词，但我始终觉得一切皆函数。至今为止，还尚未听过见过没有函数的编程语言（SQL算不算？）。

很多QoS算法都不提队列，但队列是这类算法中最重要的要素了。具体来说，QoS算法的目的就是定义一个优先级队列，定义队列元素出队列的先后顺序。

## 从预留说起

起初把Reservation翻译为下限，与上限（Limit）相对应，意为保证用户IO不低于某个值。下限两个字很容易误解，如果用户一段时间内不产生IO那实际IO为0又如何保证IO不低于某个值。它原本的含义是在系统繁忙的情况下，如果用户产生的IO高于该值，保证实际IO不低于该值；如果用户不产生IO或者产生的IO低于该值，那自然不用保证。

一次偶然机会看到有人将其翻译成预留，顿时觉得妙，至少在用户看来非常贴切。预留极易让人联想到生活中预订餐桌的情况，假如预订一个八人桌，那么餐馆的服务生就会在其中一张八人桌上放个已被预订的牌子以防止被其他顾客占用。预订张八人桌能够保证8个以内的人均有位置，但不保证第九个人也有位置，可能有也可能没有。回到存储，与预订餐桌略有区别的是，预留并不会让系统为用户腾出一部分IO处理能力而等着用户IO到来。

如何预留？
假设存储系统有两个用户，两个用户都预留了10个IOPS，并且系统IO能力超过了两个用户的预留之和。

为保证用户的预留，系统处理IO分两个阶段：第一阶段先满足每个用户的预留，第二阶段再将剩余能力分配给所有用户。第一阶段要解决的问题是如何判断预留已满足。对此，不同算法有不同的策略，最直观的当属令牌桶算法。算法每隔1/10秒就为两个用户各生成一张令牌，用户的请求只有拿到令牌后才允许出队列，当用户无令牌可用时就说明该用户已经达到预留，当所有用户都无令牌可用时代表第一阶段结束。令牌桶通过产生令牌的速度来模拟用户预留的IO平均处理速度。

mClock的策略是**依据IOPS将请求映射到时间轴，确定每个请求应该被处理的时刻。请求出队列时只要将排在当前时刻前面的请求处理完就能够满足预留要求了**。

如何确定请求应该被处理的时刻？
依据IOPS的定义，每隔1/10秒处理一个请求，也就是说，时间轴上相邻两个请求的平均间隔为1/10秒。**将系统接收到给定用户的第一个请求放到时间轴中接收到该请求的时刻所在的位置，后续请求从前个请求的位置开始向右偏移1/10秒**。假设系统在t1时刻开始接收请求，t2时刻开始处理请求，t1、t2的间隔恰好1秒。如果这段时间内系统刚好接收到10个请求，那么t2时刻处理完这10个请求就刚好满足预留；如果这段时间内接收到的请求数目超过10个，那么超过的部分将排到t2后面，t2时刻只要处理掉它前面的请求就能够满足预留要求了。

所谓将请求摆到时间轴，具体到实现层面就是为请求添加一个Tag，Tag的内容为请求应该被处理的时刻。后文将预留的Tag，称为R Tag。

考虑这样一种应用场景，用户起初有部分IO（时间段I），空闲了10分钟（时间段II）后，又有了IO（时间段III）。根据上文描述的方法，时间段III中请求的R Tag将严重滞后当前时间。这将导致时间段III中的某个时刻解决掉所有排在它前面的请求后用户实际的IOPS超过预留。这对该用户来说是件好事，但对其它用户极不公平，甚至会因为无法分配到IO而饿死。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Eq_3.png-name)

上面的公式就是解决这个问题的方法，空闲了段时间后的第1个新请求将重新被设置为接收到该请求的时刻。公式中i代表用户，r代表请求，R代表Tag的值。

前文的关注点主要在单用户如何保证预留，多用户情况最容易导致的问题是IO分配不公平。如果系统总IO能力低于两个用户的预留之和，会不会出现用户饿死的情况？因为两个用户的请求都映射到时间轴，出队列时按照时间从小到大的顺序执行，因此不会出现不公平的问题。

## 上限有何不同

没有。
预留过程可以分成两步，第一步请求入队列时为请求添加R Tag，第二步请求出队列时决定哪些请求应该出队列哪些请求不能出队列。简单来说，当前时刻前的请求出队列，当前时刻后的请求不能出队列。

上限也是如此两步，只是含义略有不同。出队列时，当前时刻前面的请求全部被处理掉代表此时IOPS已经达到用户的上限，所以不能继续处理当前时刻后面的请求，否则就超出上限了。

## 权重有何不同

有点。
上限和预留是针对给定用户的绝对值，权重是用户间的相对值。第二步只要根据P Tag从小到大的顺序依次出队列即可，即使P Tag超过了当前时间也可以继续出队列。

## IOPS、带宽和延迟

IOPS比较简单，两个请求之间没有差别。带宽稍微复杂点，要考虑到每个请求的大小。

假设某用户的预留为10MB/s，可以理解为每秒钟处理10MB数据，也可以理解为每1/10秒处理1个请求，每个请求的大小为1MB。为什么这么理解呢？因为我们需要找个参照物，此例以1MB的请求为参考，两个相邻的1MB大小的请求在时间轴上的间隔为1/10秒。那么，一个大小为512k的新请求与前一个请求的间隔应该设置为1/20；一个大小为2MB的请求和前个请求的间隔应该设置为1/5。

为什么比参照物小的请求间隔小，比参照物大的请求间隔大？这要从带宽的定义来理解。

（待续）

## 突发IO

应用场景
大文件拷贝、虚拟机的启动和迁移、数据库批量更新、页缓存刷新等。

不同厂商的定义
SolidFire定义突发IOPS能够超过上限IOPS，只能持续一小段时间。HP 3PAR定义突发IO不能超过上限，只是通过抑制其它应用来提高突发应用的优先级。mClock对突发IO的定义和HP 3PAR相同，只是提高其比例不能越过上限。

（1）只在系统有余力的情况才允许突发IO；
（2）只调整P Tag，不影响预留和上限。

dmClock先检查给定的应用是否空闲，如果空闲就给予一定的突发能力。

## 读、写和读写

（待续）

## 调度和调度引发的危机

（待续）

## 扩展mClock到多服务器

（待续）
