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
