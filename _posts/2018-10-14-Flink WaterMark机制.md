---
layout:     post
title:      Flink Water Mark
subtitle:   Flink Water Mark
date:       2018-10-14
author:     danfeng
header-img: img/post-bg-ios10.jpg
catalog: 	 true
tags:
    - Flink
    - 流处理
---       


## Flink的WaterMark
&emsp;&emsp;每个窗口都会有开始时间和结束时间（一般window的时间窗口为左闭右开的区间范围），在这段时间内，我们是否能拿到所有需要处理的数据，我们就需要watermark来配合了。

  &emsp;&emsp;Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Apache Flink Source或者自定义的Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。 Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。  Watermark的产生和Apache Flink内部处理逻辑如下图所示: 
  ![image](https://s2.ax1x.com/2019/07/20/ZzzCh8.jpg)
  
  从上文中，我们可以得出两个触发watermark的必要条件

---
- watermark时间 >= 窗口的结束时间

- 在窗口的时间范围(左闭右开)内有数据
那么，flink是如何避免数据乱流的呢？我们来看下面一张图
![image](https://s2.ax1x.com/2019/07/20/ZzzKhT.jpg)

&emsp;&emsp;这是一个典型的单通道的场景，首先我们有一个时间事件队列{2，3，1，7，3，5，9，6，12} ，两个wartermark（W4，W9）事件通过时间戳被指派给了窗口（T1-T4）。

数据流入2，3，1进入窗口，7不属于当前事件窗口，所以指派给了新窗口（T4-T8）。

数据继续流入，此时水位线到达W4，触发窗口（T1-T4）计算。

数据继续流入，9被指派给了新窗口（T9-T12）（*笔者注，这个图的事件窗口不对，个人认为是T8-T12）

## Event Time时间窗口的实现
Flink借鉴了Google的MillWheel项目，通过WaterMark来支持基于Event Time时间窗口。当操作符通过基于Event Time的时间窗口来处理数据时，它必须在确定所有属于该时间窗口的消息全部流入此操作符后，才能开始处理数据。但是由于消息可能是乱序的，所以操作符无法直接确认何时所有属于该时间窗口的消息全部流入此操作符。WaterMark包含一个时间戳，Flink使用WaterMark标记所有小于该时间戳的消息都已流入，Flink的数据源在确认所有小于某个时间戳的消息都已输出到Flink流处理系统后，会生成一个包含该时间戳的WaterMark，插入到消息流中输出到Flink流处理系统中，Flink操作符按照时间窗口缓存所有流入的消息，当操作符处理到WaterMark时，它对所有小于该WaterMark时间戳的时间窗口的数据进行处理并发送到下一个操作符节点，然后也将WaterMark发送到下一个操作符节点。


---
&emsp;&emsp;为了保证能够处理所有属于某个时间窗口的消息，操作符必须等到大于这个时间窗口的WaterMark之后，才能开始对该时间窗口的消息进行处理，相对于基于Operator Time的时间窗口，Flink需要占用更多的内存，且会直接影响消息处理的延迟时间。对此，一个可能的优化措施是，对于聚合类的操作符，可能可以提前对部分消息进行聚合操作，当有属于该时间窗口的新消息流入时，基于之前的部分聚合结果继续计算，这样的话，只需缓存中间计算结果即可，无需缓存该时间窗口的所有消息。


## 基于时间戳的排序
&emsp;&emsp;在流处理系统中，由于流入的消息是无限的，所以对消息进行排序基本上被认为是不可行的。但是在Flink流处理系统中，基于WaterMark，Flink实现了基于时间戳的全局排序。

Flink基于时间戳进行排序的实现思路如下：排序操作符缓存所有流入的消息，当其接收到WaterMark时，对时间戳小于该WaterMark的消息进行排序，并发送到下一个节点，在此排序操作符中释放所有时间戳小于该WaterMark的消息，继续缓存流入的消息，等待下一个WaterMark触发下一次排序。由于WaterMark保证了其之后不会出现时间戳比它小的消息，所以可以保证排序的正确性。需要注意的是，如果排序操作符有多个节点，只能保证每个节点的流出消息是有序的，节点之间的消息不能保证有序，要实现全局有序，则只能有一个排序操作符节点。

通过支持基于Event Time的消息处理，Flink扩展了其流处理系统的应用范围，使得更多的流处理任务可以通过Flink来执行。
