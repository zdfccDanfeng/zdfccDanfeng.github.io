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
&emsp;每个Spark executor作为一个YARN容器(container)运行。Spark可以使得多个Tasks在同一个容器(container)里面运行
