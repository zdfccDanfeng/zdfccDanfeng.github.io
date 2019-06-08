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
  -  step2: map task运行的结果是一个个的k-v对，需要发送到reduce结点进行规约操作， MapReduce提供Partitioner接口，它的作用就是根据key或value及reduce的数量来决定当前的这对输出数据最终应该交由哪个reduce task处理。默认对key求hash后再以reduce task数量取模。默认的取模方式只是为了平均reduce的处理能力，如果用户自己对Partitioner有需求，可以订制并设置到job上。
  -  step3: spill操作：包括sort ,combine（可选）操作。这一步的主要目的是由于Map task的结果经过Pattioner之后会写入到内存缓冲区中，缓冲区的作用是批量收集map结果，减少磁盘IO的影响。我们的key/value对以及Partition的结果都会被写入缓冲区。当然写入之前，key与value值都会被序列化成字节数组，整个内存缓冲区就是一个环形字节数组。内存缓冲区是有大小限制的，默认是100MB。当map task的输出结果很多时，就可能会撑爆内存，所以需要在一定条件下将缓冲区中的数据临时写入磁盘，然后重新利用这块缓冲区。这个从内存往磁盘写数据的过程被称为Spill，中文可译为溢写，字面意思很直观。这个溢写是由单独线程来完成，不影响往缓冲区写map结果的线程。溢写线程启动时不应该阻止map的结果输出，所以整个缓冲区有个溢写的比例spill.percent。这个比例默认是0.8，也就是当缓冲区的数据已经达到阈值（buffer size * spill percent = 100MB * 0.8 = 80MB），溢写线程启动，锁定这80MB的内存，执行溢写过程。Map task的输出结果还可以往剩下的20MB内存中写，互不影响。 当溢写线程启动后，需要对这80MB空间内的key做排序(Sort)。排序是MapReduce模型默认的行为，这里的排序也是对序列化的字节做的排序。另外，因为map task的输出是需要发送到不同的reduce端去，而内存缓冲区没有对将发送到相同reduce端的数据做合并，那么这种合并应该是体现是磁盘文件中的。从官方图上也可以看到写到磁盘中的溢写文件是对不同的reduce端的数值做过合并。所以溢写过程一个很重要的细节在于，如果有很多个key/value对需要发送到某个reduce端去，那么需要将这些key/value值拼接到一块，减少与partition相关的索引记录。如果client设置过Combiner，那么现在就是使用Combiner的时候了。将有相同key的key/value对的value加起来，减少溢写到磁盘的数据量。Combiner会优化MapReduce的中间结果，所以它在整个模型中会多次使用。那哪些场景才能使用Combiner呢？从这里分析，Combiner的输出是Reducer的输入，Combiner绝不能改变最终的计算结果。所以从我的想法来看，Combiner只应该用于那种Reduce的输入key/value与输出key/value类型完全一致，且不影响最终结果的场景。比如累加，最大值等。Combiner的使用一定得慎重，如果用好，它对job执行效率有帮助，反之会影响reduce的最终结果。
  - step4: merge操作：Map任务如果输出数据量很大，可能会进行好几次Spill，文件会产生很多，分布在不同的磁盘上。
  
  ![image](https://static.open-open.com/lib/uploadImg/20140521/20140521222449_800.jpg)
  <br>
  
  
  
  
  
  
      Reduce端shuffle 
  
  
#### Copy
&emsp;&emsp;Reduce 任务通过HTTP向各个Map任务拖取它所需要的数据。每个节点都会启动一个常驻的HTTP server，其中一项服务就是响应Reduce拖取Map数据。当有MapOutput的HTTP请求过来的时候，HTTP server就读取相应的Map输出文件中对应这个Reduce部分的数据通过网络流输出给Reduce。Reduce任务拖取某个Map 对应的数据，如果在内存中能放得下这次数据的话就直接把数据写到内存中。Reduce要向每个Map去拖取数据，在内存中每个Map对应一块数据，当内存 中存储的Map数据占用空间达到一定程度的时候，开始启动内存中merge，把内存中的数据merge输出到磁盘上一个文件中。如果在内存中不能放得下这个Map的数据的话，直接把Map数据写到磁盘上，在本地目录创建一个文件，从HTTP流中读取数据然后写到磁盘，使用的缓存区大小是 64K。拖一个Map数据过来就会创建一个文件，当文件数量达到一定阈值时，开始启动磁盘文件merge，把这些文件合并输出到一个文件。有些Map的数据较小是可以放在内存中的，有些Map的数据较大需要放在磁盘上，这样最后Reduce任务拖过来的数据有些放在内存中了有些放在磁盘上，最后会对这些来一个全局合并。
#### Merge Sort
&emsp;&emsp;这里使用的Merge和Map端使用的Merge过程一样。Map的输出数据已经是有序的，Merge进行一次合并排序，所谓Reduce端的 sort过程就是这个合并的过程。一般Reduce是一边copy一边sort，即copy和sort两个阶段是重叠而不是完全分开的。Reduce端的Shuffle过程至此结束。

### Spark Shuffle
&emsp;&emsp;Spark丰富了任务类型，有些任务之间数据流转不需要通过Shuffle，但是有些任务之间还是需要通过Shuffle来传递数据，比如wide dependency的group by key。Spark中需要Shuffle输出的Map任务会为每个Reduce创建对应的bucket，Map产生的结果会根据设置的partitioner得到对应的bucketId，然后填充到相应的bucket中去。每个Map的输出结果可能包含所有的Reduce所需要的数据，所以每个Map会创建R个bucket（R是reduce的 个数），M个Map总共会创建M*R个bucket。Map创建的bucket其实对应磁盘上的一个文件，Map的结果写到每个bucket中其实就是写到那个磁盘文件中，这个文件也被称为blockFile，是DiskBlockManager管理器通过文件名的Hash值对应到本地目录的子目录中创建的。每个Map要在节点上创建R个磁盘文件用于结果输出，Map的结果是直接输 出到磁盘文件上的，100KB的内存缓冲是用来创建Fast Buffered OutputStream输出流。这种方式一个问题就是Shuffle文件过多。
![image](https://static.open-open.com/lib/uploadImg/20140521/20140521222450_291.jpg)
<br>
- 首先每一个Mapper会根据Reducer的数量创建出相应的bucket，bucket的数量是MXR，其中\(M\)是Map的个数，\(R\)是Reduce的个数。
- 其次Mapper产生的结果会根据设置的partition算法填充到每个bucket中去。这里的partition算法是可以自定义的，当然默认的算法是根据key哈希到不同的bucket中去。
- 当Reducer启动时，它会根据自己task的id和所依赖的Mapper的id从远端或是本地的block manager中取得相应的bucket作为Reducer的输入进行处理。
<br>
当map端执行完毕后，会把执行的结果信息（磁盘小文件的位置，最终执行的状态等）封装到mapstatus中，然后会调用MapOutputTeackWorker向MapOutputTeackMaster上，此时Master上有所有磁盘小文件的信息，其他task想用这些数据，调用自身的MapOutputTeackWorker向主节点申请位置信息即可
接着Spark会将ArrayBuffer中的Map输出结果写入block manager所管理的磁盘中，这里文件的命名方式为： shuffle_ + shuffle_id + "_" + map partition id + "_" + shuffle partition id。
早期的shuffle write有两个比较大的问题：
Map的输出必须先全部存储到内存中，然后写入磁盘。这对内存是一个非常大的开销，当内存不足以存储所有的Map output时就会出现OOM。
每一个Mapper都会产生Reducer number个shuffle文件，如果Mapper个数是1k，Reducer个数也是1k，那么就会产生1M个shuffle文件，这对于文件系统是一个非常大的负担。同时在shuffle数据量不大而shuffle文件又非常多的情况下，随机写也会严重降低IO的性能。

   &emsp;&emsp;针对上述Shuffle 过程产生的文件过多问题，Spark有另外一种改进的Shuffle过程：consolidation Shuffle，以期显著减少Shuffle文件的数量。在consolidation Shuffle中每个bucket并非对应一个文件，而是对应文件中的一个segment部分。Job的map在某个节点上第一次执行，为每个 reduce创建bucket对应的输出文件，把这些文件组织成ShuffleFileGroup，当这次map执行完之后，这个 ShuffleFileGroup可以释放为下次循环利用；当又有map在这个节点上执行时，不需要创建新的bucket文件，而是在上次的 ShuffleFileGroup中取得已经创建的文件继续追加写一个segment；当前次map还没执行完，ShuffleFileGroup还没有 释放，这时如果有新的map在这个节点上执行，无法循环利用这个ShuffleFileGroup，而是只能创建新的bucket文件组成新的 ShuffleFileGroup来写输出。
   #### Shuffle Fetcher
&emsp;&emsp;Reduce去拖Map的输出数据，Spark提供了两套不同的拉取数据框架：通过socket连接去取数据；使用netty框架去取数据。每个节点的Executor会创建一个BlockManager，其中会创建一个BlockManagerWorker用于响应请求。当Reduce的GET_BLOCK的请求过来时，读取本地文件将这个blockId的数据返回给Reduce。如果使用的是Netty框架，BlockManager会创建ShuffleSender用于发送Shuffle数据。并不是所有的数据都是通过网络读取，对于在本节点的Map数据，Reduce直接去磁盘上读取而不再通过网络框架。Reduce拖过来数据之后以什么方式存储呢？Spark Map输出的数据没有经过排序，SparkShuffle过来的数据也不会进行排序，Spark认为Shuffle过程中的排序不是必须的，并不是所有类型的Reduce需要的数据都需要排序，强 制地进行排序只会增加Shuffle的负担。Reduce拖过来的数据会放在一个HashMap中，HashMap中存储的也是对，key是Map输出的key，Map输出对应这个key的所有value组成HashMap的value。Spark将 Shuffle取过来的每一个对插入或者更新到HashMap中，来一个处理一个。HashMap全部放在内存中。Shuffle 取过来的数据全部存放在内存中，对于数据量比较小或者已经在Map端做过合并处理的Shuffle数据，占用内存空间不会太大，但是对于比如group by key这样的操作，Reduce需要得到key对应的所有value，并将这些value组一个数组放在内存中，这样当数据量较大时，就需要较多内存。当内存不够时，要不就失败，要不就用老办法把内存中的数据移到磁盘上放着。Spark意识到在处理数据规模远远大于内存空间时所带来的不足，引入了一个具有外部排序的方案。Shuffle过来的数据先放在内存中，当内存中存储的对超过1000并且内存使用超过70%时，判断节点上可用内存如果还足够，则把内存缓冲区大小翻倍，如果可用内存不再够了，则把内存中的数据排序然后写到磁盘文件中。最后把内存缓冲区中的数据排序之后和那些磁盘文件组成一个最小堆，每次从最小堆中读取最小的数据，这个和MapReduce中的merge过程类似。

