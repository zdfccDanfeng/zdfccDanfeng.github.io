---
layout:     post
title:      Spark satge划分
subtitle:   Spark satge划分
date:       2019-01-01
author:     danfeng
header-img: img/post-bg-ios10.jpg
catalog: 	 true
tags:
    - SPARK
    - BIGDATA
--- 
  &emsp;&emsp;Spark Application只有遇到action操作时才会真正的提交任务并进行计算，DAGScheduler 会根据各个RDD之间的依赖关系形成一个DAG，
并根据ShuffleDependency来进行stage的划分，stage包含多个tasks，个数由该stage的finalRDD决定，stage里面的task完全相同，
DAGScheduler 完成stage的划分后基于每个Stage生成TaskSet，并提交给TaskScheduler，TaskScheduler负责具体的task的调度，
在Worker节点上启动task。
![img](https://upload-images.jianshu.io/upload_images/3597066-fc535e47d140e361.png?imageMogr2/auto-orient/)

```java
 /** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
  }
```
