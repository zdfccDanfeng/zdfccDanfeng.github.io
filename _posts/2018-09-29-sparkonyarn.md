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
  ![image](https://zdfccdanfeng.github.io/img/we.png)
  


##   Spark作业当调度执行流程：
- 在使用spark-summit提交spark程序后，根据提交时指定（deploy-mode）的位置，创建driver进程，driver进程根据sparkconf中的配置，初始化sparkcontext。Sparkcontext的启动后，创建DAG Scheduler（将DAG图分解成stage）和Task Scheduler（提交和监控task）两个调度模块。
- driver进程根据配置参数向resource manager（资源管理器）申请资源（主要是用来执行的executor），resource manager接到到了Application的注册请求之后，会使用自己的资源调度算法，在spark集群的worker上，通知worker为application启动多个Executor。
- executor创建后，会向resource manager进行资源及状态反馈，以便resource manager对executor进行状态监控，如监控到有失败的executor，则会立即重新创建。
- Executor会向taskScheduler反向注册，以便获取taskScheduler分配的task
- Driver完成SparkContext初始化，继续执行application程序，当执行到Action时，就会创建Job。并且由DAGScheduler将Job划分多个Stage,每个Stage 由TaskSet 组成，并将TaskSet提交给taskScheduler,taskScheduler把TaskSet中的task依次提交给Executor, Executor在接收到task之后，会使用taskRunner（封装task的线程池）来封装task,然后，从Executor的线程池中取出一个线程来执行task
- 就这样Spark的每个Stage被作为TaskSet提交给Executor执行，每个Task对应一个RDD的partition,执行我们的定义的算子和函数。直到所有操作执行完为止。如下图所示：
![image](https://upload-images.jianshu.io/upload_images/2119554-19572064cd1d37eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/462/format/webp)

&emsp;Stage的划分不仅根据RDD的依赖关系，还有一个原则是将依赖链断开，每个stage内部可以并行运行，整个作业按照stage顺序依次执行，最终完成整个Job。





  
  
