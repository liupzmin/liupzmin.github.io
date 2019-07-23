---
date: 2018-03-22 15:29
categories: 大数据
tags: 
	- spark2
	- python3
title: 在spark2.2.0中使用python 3

---

 

>Cannot run program "python3": error=2, 没有那个文件或目录



spark2中默认使用的是python2，可以通过以下三种方式之一使用python3：

1. `PYSPARK_PYTHON=python3  pyspark2`
2. 修改`~/.bash_profile`，增加 `PYSPARK_PYTHON=python3`
3. 修改`spark-env.sh`增加`PYSPARK_PYTHON=/usr/local/bin/python3`

`如果使用前2种不带绝对路径的变量声明时可能会遇到`Cannot run program "python3": error=2, 没有那个文件或目录`错误，原因是我的spark环境默认的是运行在`yarn`上的，当执行RDD任务时会在其他节点报错：`

```shell
[root@hadoop-04 ~]# PYSPARK_PYTHON=python3 pyspark2
Python 3.6.4 (default, Mar 21 2018, 13:55:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)] on linux
Type "help", "copyright", "credits" or "license" for more information.
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.2.0.cloudera2
      /_/

Using Python version 3.6.4 (default, Mar 21 2018 13:55:56)
SparkSession available as 'spark'.
>>> lines = sc.textFile('/afis/flume/auth/2018/03/16/auth.1521129675887.log')
>>> pythonlines = lines.filter(lambda line:"python" in line)
>>> pythonlines.count()
[Stage 0:>                                                          (0 + 2) / 2]18/03/22 13:25:22 WARN scheduler.TaskSetManager: Lost task 0.0 in stage 0.0 (TID 0, hadoop-02, executor 2): java.io.IOException: Cannot run program "python3": error=2, 没有那个文件或目录
        at java.lang.ProcessBuilder.start(ProcessBuilder.java:1048)
        at org.apache.spark.api.python.PythonWorkerFactory.startDaemon(PythonWorkerFactory.scala:163)
        at org.apache.spark.api.python.PythonWorkerFactory.createThroughDaemon(PythonWorkerFactory.scala:89)
        at org.apache.spark.api.python.PythonWorkerFactory.create(PythonWorkerFactory.scala:65)
        at org.apache.spark.SparkEnv.createPythonWorker(SparkEnv.scala:117)
        at org.apache.spark.api.python.PythonRunner.compute(PythonRDD.scala:128)
        at org.apache.spark.api.python.PythonRDD.compute(PythonRDD.scala:63)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:323)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:287)
        at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:87)
        at org.apache.spark.scheduler.Task.run(Task.scala:108)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:338)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
Caused by: java.io.IOException: error=2, 没有那个文件或目录
        at java.lang.UNIXProcess.forkAndExec(Native Method)
        at java.lang.UNIXProcess.<init>(UNIXProcess.java:247)
        at java.lang.ProcessImpl.start(ProcessImpl.java:134)
        at java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
        ... 14 more
......
>>> 18/03/22 13:25:23 WARN scheduler.TaskSetManager: Lost task 1.0 in stage 0.0 (TID 1, hadoop-03, executor 1): TaskKilled (stage cancelled)
```

`hadoop-02`这个节点上找不到`python3`导致任务终止,既然提示在其他节点上找不到，那在本地节点运行会是哪种结果呢？

```shell
[root@hadoop-04 ~]# PYSPARK_PYTHON=python3 pyspark2 --master local
Python 3.6.4 (default, Mar 21 2018, 13:55:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)] on linux
Type "help", "copyright", "credits" or "license" for more information.
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.2.0.cloudera2
      /_/

Using Python version 3.6.4 (default, Mar 21 2018 13:55:56)
SparkSession available as 'spark'.
>>> lines = sc.textFile('/afis/flume/auth/2018/03/16/auth.1521129675887.log')
>>> pythonlines = lines.filter(lambda line:"python" in line)
>>> pythonlines.count()
0
```

可见在本地运行是没有问题的，那问题就出在`python3`的可执行文件少了绝对路径，猜测是spark内部的任务调度执行的时候没有使用操作系统的`PATH`导致找不到可执行文件，现在把`python3`的可执行文件路径补全：

```shell
[root@hadoop-04 ~]# PYSPARK_PYTHON=/usr/local/bin/python3 pyspark2
Python 3.6.4 (default, Mar 21 2018, 13:55:56) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)] on linux
Type "help", "copyright", "credits" or "license" for more information.
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
18/03/22 14:54:52 WARN util.Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.2.0.cloudera2
      /_/

Using Python version 3.6.4 (default, Mar 21 2018 13:55:56)
SparkSession available as 'spark'.
>>> lines = sc.textFile('/afis/flume/auth/2018/03/16/auth.1521129675887.log')
>>> pythonlines = lines.filter(lambda line:"python" in line)
>>> pythonlines.count()
0                                                                               
>>> pythonlines = lines.filter(lambda line:"SessionTask" in line)
>>> pythonlines.count()
719
```

**可见，要在spark2上使用python3需要设置`PYSPARK_PYTHON`为可执行文件的绝对路径，优先推荐设置`spark-env.sh`**

```shell
[root@hadoop-01 ~]# more /etc/spark2/conf/spark-env.sh 
#!/usr/bin/env bash
##
# Generated by Cloudera Manager and should not be modified directly
##

SELF="$(cd $(dirname $BASH_SOURCE) && pwd)"
if [ -z "$SPARK_CONF_DIR" ]; then
  export SPARK_CONF_DIR="$SELF"
fi

export SPARK_HOME=/opt/cloudera/parcels/SPARK2-2.2.0.cloudera2-1.cdh5.12.0.p0.232957/lib/spark2
export DEFAULT_HADOOP_HOME=/opt/cloudera/parcels/CDH-5.14.0-1.cdh5.14.0.p0.24/lib/hadoop
export PYSPARK_PYTHON=/usr/local/bin/python3
```