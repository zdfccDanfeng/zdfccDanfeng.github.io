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
 下面从源码层面进行分析： 
```java
  // 当碰到action算子的时候，就会触发job的提交
  def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
    waiter.completionFuture.value.get match {
      case scala.util.Success(_) =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      case scala.util.Failure(exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
  
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
