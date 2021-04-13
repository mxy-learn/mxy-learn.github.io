# Hive

## 1 Hive优化
### 1.1 key分部不均匀造成的数据倾斜<br>
1、增加reduce个数<br>
2、map阶段combiner合并

        通过设置以下参数开启在Map端的聚合：
        set hive.map.aggr=true;
        相关配置参数：
        hive.groupby.mapaggr.checkinterval：
        map端group by执行聚合时处理的多少行数据（默认：100000）
        hive.map.aggr.hash.min.reduction：
        进行聚合的最小比例，预先对100000条数据做聚合，若聚合之后的数据量/100000的值大于该配置0.5，则不会聚合
        hive.map.aggr.hash.percentmemory：
        map端聚合使用的内存的最大值
        hive.map.aggr.hash.force.flush.memory.threshold：
        map端做聚合操作时hash表的最大可用内存，大于该值则会触发flush，溢写到磁盘
        hive.groupby.skewindata
        是否对GroupBy产生的数据倾斜做优化，默认为false
3、key随机数加盐<br>
4、避免使用count(distinct) 用group by替代<br>
5、hive.groupby.skewindata=true 第一步相同的key拆到不同的reduce上实现负载均衡，第二部相同的key放到同一个reduce上进行聚合操作。<br>
6、mapjoin<br>

        /*+ MAPJOIN(smallTable) */
        /*+ MAPJOIN(smallTable) */
        set hive.auto.convert.join = true;
        该参数为true时，Hive自动对左边的表统计量，如果是小表就加入内存，即对小表使用Map join
        相关配置参数：
        hive.mapjoin.smalltable.filesize;
        大表小表判断的阈值，如果表的大小小于该值则会被加载到内存中运行
        hive.ignore.mapjoin.hint；
        默认值：true；是否忽略mapjoin hint 即mapjoin标记
        hive.auto.convert.join.noconditionaltask;
        默认值：true；将普通的join转化为普通的mapjoin时，是否将多个mapjoin转化为一个mapjoin
        hive.auto.convert.join.noconditionaltask.size;
        最多可以把几个mapjoin转化为一个mapjoin

## 1.2 SQL优化

1、预处理：key尽量分布均匀；输入表列裁剪和数据过滤。<br>
2、大小表join：增加缓存表大小参数，小表进内存，map段完成join。<br>
3、JVM重用

        适用场景：
            1) 小文件个数过多
            2) task个数过多
        频繁的申请资源，释放资源降低了执行效率
        通过set mapred.job.reuse.jvm.num.tasks=n; （n为task插槽个数）来设置，道理与“池”类似。
        缺点：设置开启之后，task插槽会一直占用资源，不论是否有task运行，直到所有的task即整个job全部执行完成时，才会释放所有的task插槽资源。
4、自定义调整map/reduce数目

        详细见MapReduce
        Map数量相关的参数：
          mapred.max.split.size
          一个split的最大值，即每个map处理文件的最大值
          mapred.min.split.size.per.node
          一个节点上split的最小值
          mapred.min.split.size.per.rack
          一个机架上split的最小值
        Reduce数量相关的参数：
          mapred.reduce.tasks
          强制指定reduce任务的数量
          hive.exec.reducers.bytes.per.reducer
          每个reduce任务处理的数据量
5、大表join大表 倾斜业务数据切分，关联键分区分桶。

        建立小表
        create table table1(uid int comments '用户id',name string comment '用户名称')
        partitioned by(d string)
        clustered by(uid) sorted by(uid) into 10 buckets;
        建立大表
        create table table2(uid int comment '用户id',email string comment '用户邮箱')
        partitioned by(d string)
        clustered by(uid) sorted by(uid) into 10 buckets;
        注意：
        • 两个表关联键为uid，需要用uid分桶做排序，并且小表的分桶数是大表分桶数的倍数。
        • 对于map端连接的情况，两个表以相同方式划分桶。处理左边表内某个桶的 mapper知道右边表内相匹配的行在对应的桶内。因此，mapper只需要获取那个桶 (这只是右边表内存储数据的一小部分)即可进行连接。这一优化方法并不一定要求 两个表必须桶的个数相同，两个表的桶个数是倍数关系也可以
        • 桶中的数据可以根据一个或多个列另外进行排序。由于这样对每个桶的连接变成了高效的归并排序(merge-sort), 因此可以进一步提升map端连接的效率
        启用桶表
        set hive.enforce.bucketing = true;
        插入数据
        /**表1*/
        insert overwrite table table1 partition (d='2022-04-22')
        select uid,name from tableA where d='2022-04-22';
        /**表2*/
        insert overwrite table table2 partition (d='2022-04-22')
        select uid,email from tableB where d='2022-04-22';
        设置SMB join (Sort Merge Bucket Map join)的参数
        set hive.optimize.bucketmapjoin = true;
        set hive.optimize.bucketmapjoin.sortedmerge = true;
        set hive.input.format = org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
        查询
        select count(1)
        from (select d,uid,name from table1 where d='2022-04-22') a
        join (select d,uid,email from table2 where d='2022-04-22') b
        on a.d=b.d and a.uid=b.uid
        bucket map join 和 SMB join 的区别
        bucket map join
        • 一个表的bucket数是另一个表bucket的整数倍
        • bucket列==join列
        • 必须应用在map join场景中
        SMB join (针对bucket map join的优化)
        • 小表的bucket数=大表bucket数
        • bucket列 = join列 = sort 列
        • 必须是应用在bucket map join场景中
6、大表合并 两个大表union all合并以后group by合并行<br>
7、合并小文件

## 2 自定义函数
### 2.1 UDF
1、自定义UDF需要继承org.apache.hadoop.hive.ql.UDF。<br>
2、需要实现evaluate函数。<br>
3、evaluate函数支持重载。<br>
以下是两个数求和函数的UDF。evaluate函数代表两个整型数据相加，两个浮点型数据相加，可变长数据相加<br>
    Hive的UDF开发只需要重构UDF类的evaluate函数即可。例：<br>

        package hive.connect;
        import org.apache.hadoop.hive.ql.exec.UDF;
        public final class Add extends UDF {
        public Integer evaluate(Integer a, Integer b) {
        if (null == a || null == b) {
        return null;
                       } return a + b;
        }
        public Double evaluate(Double a, Double b) {
        if (a == null || b == null)
        return null;
        return a + b;
                       }
        public Integer evaluate(Integer... a) {
             int total = 0;
        for (int i = 0; i < a.length; i++)
        if (a[i] != null)
            total += a[i];
        return total;
                                       }
        }
4、使用步骤<br>
a）把程序打包放到目标机器上去；<br>
b）进入hive客户端，添加jar包：<br>

        hive>add jar /run/jar/udf_test.jar;
c）创建临时函数：<br>

    hive>CREATE TEMPORARY FUNCTION add_example AS 'hive.udf.Add';
d）查询HQL语句：<br>

        1. SELECT add_example(8, 9) FROM scores;
        2. SELECT add_example(scores.math, scores.art) FROM scores;
        3. SELECT add_example(6, 7, 8, 6.8) FROM scores;
e）销毁临时函数：<br>

        hive> DROP TEMPORARY FUNCTION add_example;
5、细节在使用UDF的时候，会自动进行类型转换，例如：<br>

        SELECT add_example(8,9.1) FROM scores;
注：1.   UDF只能实现一进一出的操作，如果需要实现多进一出，则需要实现UDAF<br>
### 2.2 UDAF

1、以下两个包是必须的import org.apache.hadoop.hive.ql.exec.UDAF和 org.apache.hadoop.hive.ql.exec.UDAFEvaluator。<br>
2、函数类需要继承UDAF类，内部类Evaluator实UDAFEvaluator接口。<br>
3、Evaluator需要实现 init、iterate、terminatePartial、merge、terminate这几个函数。必须要返回类型值，空的话返回null，不能为void类型<br>
a）init函数实现接口UDAFEvaluator的init函数。<br>
b）iterate接收传入的参数，并进行内部的轮转。其返回类型为boolean。<br>
c）terminatePartial无参数，其为iterate函数轮转结束后，返回轮转数据，terminatePartial类似于hadoop的Combiner。<br>
d）merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean。<br>
e）terminate返回最终的聚集函数结果。<br>

        package hive.udaf;
        import org.apache.hadoop.hive.ql.exec.UDAF;
        import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;
        public class Avg extends UDAF {
                 public static class AvgState {
                 private long mCount;
                 private double mSum;
        }
        public static class AvgEvaluator implements UDAFEvaluator {
                 AvgState state;
                 public AvgEvaluator() {
        super();
                           state = new AvgState();
                           init();
        }
        /** * init函数类似于构造函数，用于UDAF的初始化 */
        public void init() {
                 state.mSum = 0;
                 state.mCount = 0;
        }
        /** * iterate接收传入的参数，并进行内部的轮转。其返回类型为boolean * * @param o * @return */
        public boolean iterate(Double o) {
        if (o != null) {
                           state.mSum += o;
                           state.mCount++;
                 } return true;
        }
        /** * terminatePartial无参数，其为iterate函数轮转结束后，返回轮转数据， * terminatePartial类似于hadoop的Combiner * * @return */
        public AvgState terminatePartial() {
        // combiner
        return state.mCount == 0 ? null : state;
        }
        /** * merge接收terminatePartial的返回结果，进行数据merge操作，其返回类型为boolean * * @param o * @return */
        public boolean terminatePartial(Double o) {
        if (o != null) {
                           state.mCount += o.mCount;
                           state.mSum += o.mSum;
                 }
        return true;
        }
        /** * terminate返回最终的聚集函数结果 * * @return */
        public Double terminate() {
        return state.mCount == 0 ? null : Double.valueOf(state.mSum / state.mCount);
        }
        }
5、使用步骤：<br>
a）将java文件编译成Avg_test.jar。<br>
b）进入hive客户端添加jar包：

        hive>add jar /run/jar/Avg_test.jar。
c）创建临时函数：

        hive>create temporary function avg_test 'hive.udaf.Avg';
d）查询语句：

        hive>select avg_test(scores.math) from scores;
e）销毁临时函数：

        hive>drop temporary function avg_test;

### 2.3 UDTF
1、继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF。<br>
2、实现initialize, process, close三个方法<br>
UDTF首先会调用initialize方法，此方法返回UDTF的返回行的信息（返回个数，类型）。<br>
初始化完成后，会调用process方法，对传入的参数进行处理，可以通过forword()方法把结果返回。最后close()方法调用，对需要清理的方法进行清理。<br>
下面是我写的一个用来切分”key:value;key:value;”这种字符串，返回结果为key, value两个字段。供参考：

        import java.util.ArrayList;
        import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
        import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
        import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
        import org.apache.hadoop.hive.ql.metadata.HiveException;
        import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
        import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
        import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
        import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

           public class ExplodeMap extends GenericUDTF{

               @Override
               public void close() throws HiveException {
        // TODO Auto-generated method stub
               }

               @Override
               public StructObjectInspector initialize(ObjectInspector[] args)
                       throws UDFArgumentException {
        if (args.length != 1) {
        throw new UDFArgumentLengthException("ExplodeMap takes only one argument");
                   }
        if (args[0].getCategory() != ObjectInspector.Category.PRIMITIVE) {
        throw new UDFArgumentException("ExplodeMap takes string as a parameter");
                   }

                   ArrayList<String> fieldNames = new ArrayList<String>();
                   ArrayList<ObjectInspector> fieldOIs = new ArrayList<ObjectInspector>();
                   fieldNames.add("col1");
                   fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
                   fieldNames.add("col2");
                   fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);

        return ObjectInspectorFactory.getStandardStructObjectInspector(fieldNames,fieldOIs);
               }

              @Override
               public void process(Object[] args) throws HiveException {
        String input = args[0].toString();
        String[] test = input.split(";");
        for(int i=0; i<test.length; i++) {
        try {
        String[] result = test[i].split(":");
                           forward(result);
                       } catch (Exception e) {
        continue;
                      }
                 }
               }
           }
3、使用步骤<br>
UDTF有两种使用方法，一种直接放到select后面，一种和lateral view一起使用。<br>
- 直接select中使用：select explode_map(properties) as (col1,col2) from src;
不可以添加其他字段使用：select a, explode_map(properties) as (col1,col2) from src
不可以嵌套调用：select explode_map(explode_map(properties)) from src
不可以和group by/cluster by/distribute by/sort by一起使用：select explode_map(properties) as (col1,col2) from src group by col1, col2
- 和lateral view一起使用：select src.id, mytable.col1, mytable.col2 from src lateral view explode_map(properties) mytable as col1, col2;
此方法更为方便日常使用。执行过程相当于单独执行了两次抽取，然后union到一个表里。

## 3 知识点
### 3.1 Hive排序
- Order By 对于查询结果做全排序，只允许有一个reduce处理。当数据量较大时，应慎用。严格模式下，必须结合limit来使用
- Sort By 对于单个reduce的数据进行排序
- Distribute By 分区排序，经常和Sort By结合使用
- Cluster By 相当于 Sort By + Distribute By，Cluster By不能通过asc、desc的方式指定排序规则；
- 可通过 distribute by column sort by column asc|desc 的方式结合使用。

### 3.2 严格模式
- 对于分区表，必须添加where对于分区字段的条件过滤；
- order by语句必须包含limit输出限制；
- 限制执行笛卡尔积的查询。

### 3.3 并行计算
hive执行sql默认是顺序执行，如下sql如果使用并行计算会大大提高效率，但是集群压力也增大：

        select wc.col1,bk.col2 from
        (select count(*) as col1 from wordcount) wc,
        (select count(*) as col2 from bucket) bk;
- 两条子查询可以使用并行计算
- 通过设置以下参数开启并行模式：

        set hive.exec.parallel=true;
**注意：**
hive.exec.parallel.thread.number
代表一次SQL计算中允许并行执行的job个数的最大值