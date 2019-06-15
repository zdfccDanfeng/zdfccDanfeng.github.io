---
layout:     post
title:      
subtitle:   Spark Lineage探讨
date:       2018-12-15
author:     danfeng
header-img: img/post-bg-ios10.jpg
catalog: 	 true
tags:
    - SPARK
    - BIGDATA
--- 



###  Spark lineage

&emsp;&emsp;利用内存加快数据加载，在其它的In-Memory类数据库或Cache类系统中也有实现。Spark的主要区别在于它采用血统（Lineage）
来时实现分布式运算环境下的数据容错性（节点失效、数据丢失）问题。RDD Lineage被称为RDD运算图或RDD依赖关系图，是RDD所有父RDD的图。它是在RDD上执行transformations函数并创建逻辑执行计划（logical execution plan）的结果，是RDD的逻辑执行计划。相比其它系统的细颗粒度的内存数据更新级别的备份或者LOG机制，RDD的Lineage记录的是粗颗粒度的特定数据转换（Transformation）操作（filter, map, join etc.)行为。当这个RDD的部分分区数据丢失时，它可以通过Lineage找到丢失的父RDD的分区进行局部计算来恢复丢失的数据，这样可以节省资源提高运行效率。这种粗颗粒的数据模型，限制了Spark的运用场合，但同时相比细颗粒度的数据模型，也带来了性能的提升。


### 依赖的类型

  依赖关系决定Lineage的复杂程度，同时也是的RDD具有了容错性。因为当某一个分区里的数据丢失了，Spark程序会根据依赖关系进行局部计算来恢复丢失的数据。依赖的关系主要分为2种，分别是 宽依赖（Wide Dependencies）和窄依赖（Narrow Dependencies）。

### 什么是宽依赖

  宽依赖：是指子RDD的分区依赖于父RDD的多个分区或所有分区，也就是说存在一个父RDD的一个分区对应一个子RDD的多个分区。

### 什么是窄依赖

  窄依赖：是指父RDD的每一个分区最多被一个子RDD的分区所用，表现为一个父RDD的分区对应于一个子RDD的分区或多个父RDD的分区对应于一个子RDD的分区，也就是说一个父RDD的一个分区不可能对应一个子RDD的多个分区。

  判断依赖的本质：判断是宽依赖还是窄依赖的本质实际上是根据父RDD的分区和对应的子RDD的分区来进行区分宽依赖和窄依赖的。当父RDD的分区对应多个分区时，也就是说父RDD的分区对应的另一部分数据可能是其他子RDD的数据，那么当该RDD数据丢失后，进行容错从分区就会把该RDD跟另外一个RDD的数据都重新计算一遍，这样就会导致冗余计算（因为，另一个子RDD的数据是当前丢失的子RDD所不需要的也计算了一遍）。

  对于 Shuffle Dependencies（一般是宽依赖），Stage 计算的输入和输出在不同的节点上，输入节点的Lineage完好，而输出节点死机的情况，通过重新计算恢复数据这种情况下，这种方法容错是有效的，否则无效，因为无法重试，需要向上追溯其祖先看是否可以重试（这就是 lineage，血统的意思），窄依赖对于数据的重算开销要远小于 宽依赖的数据重算开 销。

  窄依赖（ Narrow Dependency） 和 Shuffle Dependency 的概念主要用在两个地方：一个是容错中相当于 Redo 日志的功能；另一个是在调度中构建 DAG 作为不同 Stage 的划分点。
