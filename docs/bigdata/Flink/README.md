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
### 4.1 任务调度原理

![avatar](f2.png)
1. 客户端不是运行时和程序执行的一部分，但它用于准备并发送dataflow(JobGraph)给Master(JobManager)，然后，客户端断开连接或者维持连接以等待接收计算结果。而Job Manager会产生一个执行图(Dataflow Graph)
2. 当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。
3. Client 为提交 Job 的客户端，可以是运行在任何机器上（与 JobManager 环境连通即可）。提交 Job 后，Client 可以结束进程（Streaming的任务），也可以不结束并等待结果返回。
4. JobManager 主要负责调度 Job 并协调 Task 做 checkpoint，职责上很像 Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行。
5. TaskManager 在启动的时候就设置好了槽位数（Slot），每个 slot 能启动一个 Task，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 Netty 连接，接收数据并处理。

>注：如果一个Slot中启动多个线程，那么这几个线程类似CPU调度一样共用同一个slot

### 4.2 Slots和并行度
![avatar](f3.png)
![avatar](f4.png)

flink中的Slot可以理解为spark中的executor<br>
是否分组的区别就在于，同一个链路上前后不同的算子能否公用一个Slot，默认是可以的，如果设置了分组就不能公用会像上面图二一样全部摊平。<br>
正常情况下，采用默认不设置分组，各个数据链路互不影响，健壮性更高，同时资源使用更均衡。

### 4.3 程序和数据流