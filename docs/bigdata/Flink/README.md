# Flink

## 1 特点
- 事件驱动（Event-driven）
- 基于流处理，一切皆由流组成，离线数据是有界的流；实时数据是一个没有界限的流。（有界流、无界流）

## 2 Flink vs Spark Streaming
- 数据模型<br>
Spark采用RDD模型，spark streaming的DStream实际上也就是一组组小批数据RDD的集合<br>
flink基本数据模型是数据流，以及事件（Event）序列<br>
- 运行时架构<br>
spark是批计算，将DAG划分为不同的stage，一个完成后才可以计算下一个<br>
flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点处理<br>

## 3 任务提交

```./flink run -c com.atguigu.wc.StreamWordCount FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port 7777```

**任务提交流程**

![avatar](f1.png)
1. Flink任务提交后，Client向HDFS上传Flink的Jar包和配置
2. 之后客户端向Yarn ResourceManager提交任务，ResourceManager分配Container资源并通知对应的NodeManager启动ApplicationMaster
3. ApplicationMaster启动后加载Flink的Jar包和配置构建环境，去启动JobManager，之后JobManager向Flink自身的RM进行申请资源，自身的RM向Yarn 的ResourceManager申请资源(因为是yarn模式，所有资源归yarn RM管理)启动TaskManager
4. Yarn ResourceManager分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager
5. NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager，TaskManager启动后向JobManager发送心跳包，并等待JobManager向其分配任务。

## 4 运行原理
