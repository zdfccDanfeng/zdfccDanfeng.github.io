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
在正式的分析SparkShuffle之前，先梳理下MR的shuffle流程。 Shuffle是连接Map 任务和Task任务的纽带。
shuffle 开始和结束时间：开始时间：map执行完成有输出文件产生，shuffle开始；结束时间：reduce输入文件最终确定了，shuffle结束；

