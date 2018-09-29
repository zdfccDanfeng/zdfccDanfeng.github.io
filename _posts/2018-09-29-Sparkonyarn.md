---
layout:     post
title:      Spark on Yarn模式探讨
subtitle:   Spark on Yarn模式探讨
date:       2018-09-29
author:     danfeng
header-img: img/post-bg-ios10.jpg
catalog: 	 true
tags:
    - HADOOP
    - BIGDATA
--- 




# spark on yarn探讨
&emsp;每个Spark executor作为一个YARN容器(container)运行。Spark可以使得多个Tasks在同一个容器(container)里面运行。当在YARN上运行Spark作业，每个Spark executor作为一个YARN容器(container)运行。Spark可以使得多个Tasks在同一个容器(container)里面运行。这是个很大的优点注意这里和Hadoop的MapReduce作业不一样，MapReduce作业为每个Task开启不同的JVM来运行。虽然说MapReduce可以通过参数来配置。详见mapreduce.job.jvm.numtasks。从广义上讲，yarn-cluster适用于生产环境；而yarn-client适用于交互和调试，也就是希望快速地看到application的输出。
在我们介绍yarn-cluster和yarn-client的深层次的区别之前，我们先明白一个概念：Application Master。在YARN中，每个Application实例都有一个Application Master进程，它是Application启动的第一个容器。它负责和ResourceManager打交道，并请求资源。获取资源之后告诉NodeManager为其启动container。
## yarn-cluster和yarn-client模式的区别
&emsp;&emsp;从深层次的含义讲，yarn-cluster和yarn-client模式的区别其实就是Application Master进程的区别，yarn-cluster模式下，driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行。然而yarn-cluster模式不适合运行交互类型的作业。而yarn-client模式下，Application Master仅仅向YARN请求executor，client会和请求的container通信来调度他们工作，也就是说Client不能离开。看下下面的两幅图应该会明白（上图是yarn-cluster模式，下图是yarn-client模式）

##    下图给出具体区别：上图是Yarn-cluster模式，下图是Yarn-client模式
   ![image](https://zdfccdanfeng.github.io/img/spark-yarn-f31.png)

![image](https://zdfccdanfeng.github.io/img/spark-yarn-f22.png)

&emsp;对spark而言：Job=多个stage，Stage=多个同种task, Task分为ShuffleMapTask和ResultTask，Dependency分为ShuffleDependency和NarrowDependency
  具体图示如下：
  ![image](https://images2015.cnblogs.com/blog/1004194/201608/1004194-20160829182313371-1648664691.png)

