---
layout:     post
title:      SparkShuffle和MapReduceShuffle的对比
subtitle:   SparkShuffle和MapReduceShuffle的对比.
date:       2018-10-12
author:     danfeng
header-img: img/post-bg-ios10.jpg
catalog: 	 true
tags:
    - HADOOP
    - BIGDATA
    - SPARK
--- 

### MR Shuffle
&emsp;&emsp;在正式的分析SparkShuffle之前，先梳理下MR的shuffle流程。 Shuffle是连接Map 任务和Task任务的纽带。Shuffle的本义是洗牌、混洗，把一组有一定规则的数据尽量转换成一组无规则的数据，越随机越好。MapReduce中的Shuffle更像是洗牌的逆过程，把一组无规则的数据尽量转换成一组具有一定规则的数据。
为什么MapReduce计算模型需要Shuffle过程？我们都知道MapReduce计算模型一般包括两个重要的阶段：Map是映射，负责数据的key分发；Reduce是规约，负责数据的计算归并。Reduce的数据来源于Map，Map的输出即是Reduce的输入，Reduce需要通过 Shuffle来获取数据。
&emsp;&emsp;在Hadoop这样的集群环境中，大部分map task与reduce task的执行是在不同的节点上。当然很多情况下Reduce执行时需要跨节点去拉取其它节点上的map task结果。如果集群正在运行的job有很多，那么task的正常执行对集群内部的网络资源消耗会很严重。这种网络消耗是正常的，我们不能限制，能做的就是最大化地减少不必要的消耗。还有在节点内，相比于内存，磁盘IO对job完成时间的影响也是可观的。从最基本的要求来说，我们对Shuffle过程的期望可以有： 

- 完整地从map task端拉取数据到reduce 端。
- 在跨节点拉取数据时，尽可能地减少对带宽的不必要消耗。
- 减少磁盘IO对task执行的影响。

从Map输出到Reduce输入的整个过程可以广义地称为Shuffle。Shuffle横跨Map端和Reduce端，在Map端包括Spill过程，在Reduce端包括copy和sort过程
 ![image](https://static.open-open.com/lib/uploadImg/20140521/20140521222449_182.jpg)。首先界定下shuffle在整个MR操作过程中的边界：
  开始时间：map执行完成有输出文件产生，shuffle开始；结束时间：reduce输入文件最终确定了，shuffle结束；
  
  
      Map端shuffle 
  
 &emsp;&emsp;每个map task都有一个内存缓冲区，存储着map的输出结果，当缓冲区快满的时候需要将缓冲区的数据以一个临时文件的方式存放到磁盘，当整个map task结束后再对磁盘中这个map task产生的所有临时文件做合并，生成最终的正式输出文件，然后等待reduce task来拉数据。
<br>![image](http://dl.iteye.com/upload/attachment/456529/641c4f01-6c9d-322c-b428-9981866d86a6.jpg)
  -  step1: 根据输入文件的split数目 ，创建map task，split的数目和map task的数目一一对应。
  -  step2: map task运行的结果是一个个的k-v对，需要发送到reduce结点进行规约操作， MapReduce提供Partitioner接口，它的作用就是根据key或value及reduce的数量来决定当前的这对输出数据最终应该交由哪个reduce task处理。默认对key hash后再以reduce task数量取模。默认的取模方式只是为了平均reduce的处理能力，如果用户自己对Partitioner有需求，可以订制并设置到job上。
  -  step3: spill操作：包括sort ,combine（可选） ,merge操作。这一步的主要目的是由于Map task的结果经过Pattioner之后会写入到内存缓冲区中，缓冲区的作用是批量收集map结果，减少磁盘IO的影响。我们的key/value对以及Partition的结果都会被写入缓冲区。当然写入之前，key与value值都会被序列化成字节数组，整个内存缓冲区就是一个环形字节数组。
