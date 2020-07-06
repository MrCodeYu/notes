

# 1. Flink核心概念和编程模型

## 1.1 flink生态的核心组件栈的分层

![image-20200629154620075](./image-20200629154620075.png)



## 1.2 flink api的分层

SQL -> High-Level Language

table api  ->  declarative DSL

DataStream/DataSet  ->  core APIs

Stateful Stream Processing  ->  Low-level building block(stream, state, event time)

注意：

（1） 越底层API越灵活，越往上层API越轻便

（2） SQL的必定存在解析，和专称mr的过程，必定有性能损失

（3） SQL构建在table上，相当于是一章虚拟表

（4） 不同类型的table构建在不痛的table环境

（5） table可以和DataStream或DataSet互相转换

（6） streaming sql不同于存储的sql，最终会变成流式执行计划



## 1.3 DataFlow的基本套路

![image-20200629160626481](./image-20200629160626481.png)

1. 构建安装环境，stream或batch
2. 创建source
3. 对数据transformations
4. 输出结果sink



## 1.4 并行化DataFlow

![image-20200630094442017](./image-20200630094442017.png)

实际上在设置并行度后，算子会并行运行。



## 1.5 数据的传递

one-to-one：同spark的单依赖

多依赖：

1. 改变流的分区
2.  重新分区策略取决于使用算子

keyby() -> repartition by hash key，根据hash key来发给分区

broadcast -> 给每个机器都发一份

rebalance() -> repartition randomly



## 1.6 windows

1. count windows

   根据消息的条数来划分的窗口

2. Time windows

   `Tumbling window` 翻滚窗口

   `Sliding window` 滑动窗口

   `Session Window` 

3. 自定义window



## 1.7 各种Time

1. Event Time 事件时间

   event产生的时间，比如hdfs日志里的时间

2. Ingestion Time 摄取时间

   日志进入到flink source的时间

3. Processing Time 处理时间

   日志被算子处理的时间

![image-20200630094401958](./image-20200630094401958.png)



## 1.8 Stateful Operations

1. 状态

state一般指具体的task/operator的状态

2. Operator state

某个operator处理到数据的状态，每个并发都有自己的状态

3. Keyed state

基于KeyedStream上的状态。这个状态和key绑定，对keyedstream每个key都有一个state

4. 原始状态和托管状态

待后面补充

5. state backend

The exact data structures in which the key/values indexes are stored depends on the chosen [state backend](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/state_backends.html). One state backend stores data in an in-memory hash map, another state backend uses [RocksDB](http://rocksdb.org/) as the key/value store. In addition to defining the data structure that holds the state, the state backends also implement the logic to take a point-in-time snapshot of the key/value state and store that snapshot as part of a checkpoint.

<img src="./image-20200630103236640.png" alt="image-20200630103236640" style="zoom:50%;" />



## 1.9 Checkpoint

1. checkpoint

在某个时刻，将所有task的状态做快照，存储到state backend

2. 轻量级容错
3. 保证exactly-once
4. 用于内部失败的恢复
5. 基本原理

-  source注入barrier

- barrier作为checkpoint的标志

6. 无需人工干预



## 1.10 Savepoint

1. 功能和checkpoint相同，用于存储某个时刻task的状态
2. 两种触发方式

- Cancel with savepoint

- 手动出发

3. savepoint是特殊的checkpoint，不会过期，不会覆盖，除非手动删除。



# 2. Flink运行时架构

- client
- The **JobManagers** (also called *masters*)功能是协调分布式的执行。调度task, 协调 checkpoints, 协调错误时恢复等。

There is always at least one Job Manager. A high-availability setup will have multiple JobManagers, one of which one is always the *leader*, and the others are *standby*.

- The **TaskManagers** (also called *workers*) 功能是执行dataflow的task (或具体点是执行子任务)，并且缓存和交换数据

There must always be at least one TaskManager.

- JobManager和TaskManager可以多种方式运行，比如standalone，YARN，Mesos等
- 角色通信：Akka
- 数据传输：Netty

## 2.1 flink standalone运行时架构

<img src="./image-20200630092820669.png" alt="image-20200630092820669" style="zoom:50%;" />

## 2.2 flink yarn运行时架构

![image-20200630091330427](./image-20200630091330427.png)

1. client确认RS是否有足够资源（启动ApplicationMaster的memory和vcores），资源充足时将jar包和配置上传到HDFS
2. client请求RS分配一个container用于启动ApplicationMaster
3. 当client将AM的配置和jar文件当作资源向RS注册之后，RS会分配一个container并指定一个NM启动AM。Flink JobManager和AM运行在相同container
4. 当JobManager启动完成后，AM就知晓JM的地址（他自己）。AM会生成Flink TaskManager的配置，并上传到HDFS
5. AM开始分配Flink TaskManager的container，TaskManager会去HDFS上下载jar包和修改后的配置，当TM启动完成后Flink开始接收Jobs

## 2.3 TaskManager slot && 共享 slot

<img src="./image-20200630103526049.png" alt="image-20200630103526049" style="zoom:50%;" />



- 每个worker（TaskManager）都是一个JVM进程，每个task slot是一个线程，task slot运行在JM中，task slot决定了worker接受多少task。

- 每个task slot都是TaskManager资源的固定子集。例如一个TaskManager有6G内存，包含3个task slot，那么每个task slot有2G内存。目前cpu没做资源隔离，只有内存隔离。

- 同一个TaskManager（同一个JVM）中运行的task共享TCP链接、心跳、数据集和数据结构，可以减少任务开销



<img src="./image-20200630110543126.png" alt="image-20200630110543126" style="zoom:50%;" />

默认情况下，flink允许同一个job的subtask共享slot，好处如下：

- flink集群的最大并行度等于slot数（前提是都在同一个slot sharing group）
- 如果不共享slot，资源费密集型操作如map会使用和资源密集型操作window()一样多的资源，但实际上map并不需要那么多资源。如图通过共享slot，我们将并行度从2提升到6来充分利用slot，同时确保资源开销大的subtask可以公平分布在TaskManager中。

## 2.4 Operator Chain && Task

为了高效执行任务，flink尽可能把operator的subtask chain在一起形成新的task

![image-20200630165452427](./image-20200630165452427.png)

1. Operator Chain的优点：

- 减少线程的切换（cpu是分时段的，多线程轮流运行，所以有切换）
- 减少缓冲区的开销
- 提高整体吞吐量并且减少延迟
- 减小序列化和反序列化（写到缓冲区和从缓冲区取需要序列化和反序列化）

2. Operator Chain的组成条件（缺一不可）

- 算子没有禁用Chain
- 上下游operator并行度一致
- 下游算子入度是1
- 上下游算子在同一个slot group（会通过slot group控制分配到同一个slot）
- 下游节点的chain策略为ALWAYS
- 上游节点chain的策略为ALWAYS或HEAD（表示可以和下游连）
- 上下游算子之间不是shuffle

## 2.5 编程改变OperatorChain的行为

- 在DataStream的opeartor后调用startNewChain()，表示这个operator开始是新的chain

- disableChaining()来指定算子不参与chain

  ```java
  source.flatMap(new CountWithOperatorListState()).disableChaining()
  ```

- 通过改变slot group来改变chain行为

  ```java
  source.flatMap(new CountWithOperatorListState()).disableChaining()
  ```

- 调整并行度

## 2.6 SlotSharingGroup（soft）&& CoLocationGroup(hard)

1. slotSharingGroup（软限制）

- 在不同slot group的task不会共享同一个slot

- 保证同一个group的并行度相同的sub-tasks共享同一个slot

- 算子默认的group是default

- 为了防止不合理共享slot，用户可以通过slotSharingGroup("name")强制指定共享组，从而改变共享slot行为

  ```java
  source.flatMap(new CountWithOperatorListState()).slotSharingGroup("group1")
  ```

- 下游算子group怎么确定？如果自身没有设置group，那就和上游算子相同
- 可以适当减少每个slot运行的线程数量，从而整体上减小机器负载

2. CoLocationGroup（强制）

- 保证所有并行度相同的sub-task运行在同一个slot
- 主要用于迭代流（flink ml）

## 2.7 slot和parallelism的关系

1. 如果所有operator处于同一个slot group，所需的task slots数量和task最高并行度相同

2. 如果operator有多个slot group，所需的slot是各个group最大并行度之和。下图任务需要slot为10+20=30

   <img src="./image-20200701105557203.png" alt="image-20200701105557203" style="zoom:50%;" />



# 3. Flink 编程

## 3.1 DataSet和DataStream

- 表示flink分布式数据集

- 包含重复的、不可变数据集

- DataSet有界，DataSteam可以无界

- 可以从数据源、各类算子操作创建

## 3.2 指定key

例如 join,coGroup, keyBy, groupBy, Reduce, GroupReduce,Windows都需要指定key

1. Tuple定义key

```java
DataStreamSource<Tuple3<String, Integer, Integer>> source = env.fromElements(
        new Tuple3<String, Integer, Integer>("a", 1, 2),
        new Tuple3<String, Integer, Integer>("b", 2, 3)
);

// 元组第一个元素为key
source.keyBy(0);
// 元组第一和二个元素为key
source.keyBy(0, 1);
```

2. Java实例key选择

举例有两个类：

```java
class Person {
    public Person(String name, int age, Address address){
        this.name = name;
        this.address = address;
        this.age = age;
    }
    String name;
    int age;
    Address address;

}

class Address {
    public Address(String city, String street, Tuple2<String, Integer> home) {
        this.city = city;
        this.street = street;
        this.home = home;
    }
    String city;
    String street;
    Tuple2<String, Integer> home;
}
```

在选取属性时：

```
// 以name为key
personInfoDS.keyBy("name");
// 以Person中Address的city为key
personInfoDS.keyBy("address.city");
// 以Person中Address的home的第一个元素为key
personInfoDS.keyBy("address.home._0");
// 用keySelector选择key
personInfoDS.keyBy(new KeySelector<Person, String>() {
    @Override
    public String getKey(Person person) throws Exception {
        return person.address.city + "_" + person.age;
    }
});
```

## 3.3 计数器和累加器

- 计数器

```java
DataStreamSource<String> source = env.fromElements("a", "b", "a", "c");

DataStream<Tuple2<String, Integer>> wordCount =
        source.map(new RichMapFunction<String, Tuple2<String, Integer>>() {
						// 在Rich方法中可用，自定义累加器
            IntCounter lineCounter = new IntCounter();

            @Override
            public void open(Configuration parameters) throws Exception {
                super.open(parameters);
                // open方法中注册累加器
                getRuntimeContext().addAccumulator("line_counter", lineCounter);
            }

            @Override
            public Tuple2<String, Integer> map(String value) throws Exception {
                // 在要计数的地方累加
                lineCounter.add(1);
                return new Tuple2<>(value, 1);
            }
        });

// execute会将累加器的结果保存在JobExecutionResult
JobExecutionResult executionResult = env.execute("counter");
// 获取累加器
Integer counter = executionResult.getAccumulatorResult("line_counter");
System.out.println(counter);
```

- 累加器

自定义累加器

待补充

## 3.4 DataStream APIs

### 3.4.1 Flink的四层执行计划

<img src="./image-20200702103944671.png" alt="image-20200702103944671" style="zoom:40%;" />

### 3.4.2 Flink生成的Graph

1. StreamGraph

- 根据代码生成的graph，表示所有operator的拓扑结构
- 在client端生成
- 在StreamExecutionEnvironment.execute()中调用，将所有算子存储在List<StreamTransformation<?>> transformations中。transformations描述DataStream之间的转换关系和StreamNode和StreamEdge等信息

2. JobGraph

- 优化streamGraph
- 将operator chain在一起
- 在client端生成
- StreamNode变成JobVertex，StreamEdge变成JobEdge；配置checkpoint策略；配置重启策略；根据group指定JobVertex所属的SlotSharingGroup

3. ExecutionGraph

- 对JobGraph进行并行化
- 在JobManager端生成
- JobVertxt变成ExecutionJobVertex，JobEdge变成ExecutionEdge;ExecutionJobVertex并发任务；JobGraph是二维结构，根据二位结构分发对应Vertext到指定slot

4. 物理执行计划

- 实际执行图，不可见

<img src="./image-20200702105737430.png" alt="image-20200702105737430" style="zoom:50%;" />

![image-20200702105758837](./image-20200702105758837.png)

### 3.4.3 Source

Source是flink的数据源，有内置数据源也支持用户自定义

1. 基于文件

- readTextFile(path)
- readFile(fileInputFormat, path)
- readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)

2. 基于socket

- socketTextStream

3. 基于Collection

- fromCollection(Collection)
- fromCollection(Iterator, Class)：Class是Iterator中数据类型
- fromElements(T ...)
- fromParallelCollection(SplittableIterator, Class)
- generateSequence(from, to)

4. 用户自定义

- ​	继承SourceFunction
- 继承RichSourceFunction
- 继承ParellelSourceFunction
- addSource(new CustomSource)调用

代码示例：自定义Mysql数据源，继承RichSourceFunction

```java
public class MysqlSource extends RichSourceFunction<HashMap<String, Tuple2<String, Integer>>> {

    private String jdbcUrl = "jdbc:mysql://127.0.0.1:3306/test";
    private String user = "root";
    private String passwd = "root";
    private Integer secondInterval = 5;

    private Connection conn = null;
    private PreparedStatement pst1 = null;
    private PreparedStatement pst2 = null;

    private boolean isRunning = true;
    private boolean isFirstTime = true;

    public MysqlSource(){}

    public MysqlSource(String jdbcUrl, String user, String passwd,Integer secondInterval) {
        this.jdbcUrl = jdbcUrl;
        this.user = user;
        this.passwd = passwd;
        this.secondInterval = secondInterval;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        Class.forName("com.mysql.jdbc.Driver");
        conn = DriverManager.getConnection(jdbcUrl, user, passwd);

    }

  /**
    * 当my_status中up_status更新后，才查询account的数据并发送
    **/
    @Override
    public void run(SourceContext<HashMap<String, Tuple2<String, Integer>>> out) throws Exception {
        String staticStatusSql = "SELECT up_status FROM my_status";
        String sql = "SELECT id,name,age FROM account";

        pst1 = conn.prepareStatement(staticStatusSql);
        pst2 = conn.prepareStatement(sql);

        HashMap<String, Tuple2<String, Integer>> mysqlData = new HashMap();

        while (isRunning){

            ResultSet rs1 = pst1.executeQuery();
            Boolean isUpdateStatus = false;
            while(rs1.next()){
                isUpdateStatus = (rs1.getInt("up_status") == 1);
            }
            System.out.println();
            System.out.println("isUpdateStatus:  " + isUpdateStatus + " isFirstTime: " + isFirstTime);

            if(isUpdateStatus || isFirstTime){
                ResultSet rs2 = pst2.executeQuery();
                while(rs2.next()){
                    int id = rs2.getInt("id");
                    String name = rs2.getString("name");
                    int age = rs2.getInt("age");
                    mysqlData.put(id + "", new Tuple2<String, Integer>(name, age));
                }
                isFirstTime = false;
                System.out.println("我查了一次mysql，数据是： " + mysqlData);
                out.collect(mysqlData);
            }else{
                System.out.println("这次没查mysql");
            }

            Thread.sleep(secondInterval * 1000);
        }
    }

    @Override
    public void cancel() {

        isRunning = false;
    }

    @Override
    public void close() throws Exception {
        super.close();
        if(conn != null){
            conn.close();
        }
        if(pst1 != null){
            pst1.close();
        }
        if(pst2 != null){
            pst2.close();
        }
    }
}
```

它处调用：

```java
env.addSource(new MysqlSource())
```

### 3.4.4 Sink

- 继承SinkFunction
- 继承RichSinkFunction
- 从addSink(new CustomSink)调用

代码示例，自定义mysql sink，继承RichSinkFunction

```java
public class MysqlSink extends RichSinkFunction<Tuple3<Integer, String, Integer>> {

    private String jdbcUrl = "jdbc:mysql://127.0.0.1:3306/test";
    private String user = "root";
    private String passwd = "root";

    private Connection conn = null;
    private PreparedStatement pst = null;
    private PreparedStatement pstIst = null;

    public MysqlSink(){}

    public MysqlSink(String jdbcUrl, String user, String passwd) {
        this.jdbcUrl = jdbcUrl;
        this.user = user;
        this.passwd = passwd;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        Class.forName("com.mysql.jdbc.Driver");
        conn = DriverManager.getConnection(this.jdbcUrl, this.user, this.passwd);
    }

    @Override
    public void invoke(Tuple3<Integer, String, Integer> student, Context context) throws Exception {
        String creatTableSQL = "CREATE TABLE IF NOT EXISTS test.mysql_sink_test" +
                "(" +
                "id INT," +
                "stu_name VARCHAR(25)," +
                "age INT" +
                ")";

        String insertSQL =
                "INSERT INTO test.mysql_sink_test(id,stu_name,age) VALUES(?,?,?)";

        // 如果表不存在，建表
        pst = conn.prepareStatement(creatTableSQL);
        pst.execute();

        // 插入语句
        pstIst = conn.prepareStatement(insertSQL);
        pstIst.setInt(1, student.f0);
        pstIst.setString(2, student.f1);
        pstIst.setInt(3, student.f2);
        int result = pstIst.executeUpdate();
        System.out.println(result);
    }

    @Override
    public void close() throws Exception {
        super.close();
        pst.close();
        pstIst.close();
        conn.close();
    }
}
```

调用方法：

```java
source.addSink(new MysqlSink());
```

- 收集sink的数据

```java
// 方法1:print
lessThanZero.print();

// 方法2:DataStreamUtils.collect方法
Iterator<Long> results = DataStreamUtils.collect(lessThanZero);
while(results.hasNext()){
  System.out.println(results.next());
}
```

### 3.4.5 Operator

Please see [operators](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/operators/index.html) for an overview of the available stream transformations.

![image-20200703105155347](/Users/yuxiang/学习资料/Flink实战课程(1)/flink资料整理/image-20200703105155347.png)

| Transformation                                               | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| **Map** <br />DataStream → DataStream                        | 一进一出                                                     |
| **FlatMap**<br />DataStream → DataStream                     | Takes one element and produces zero, one, or more elements. A flatmap function that splits sentences to words |
| **Filter** <br />DataStream → DataStream                     | 过滤操作                                                     |
| **KeyBy** <br />DataStream→KeyedStream                       | 根据key将不同的日志分配到不同的partition中，内部是根据hash分配key<br />注意：要使用keyBy必须：<br />1. POJO类重写`hashCode()`<br />2. 不能是array<br /><br />`source.keyBy(0);` |
| **Reduce** <br />KeyedStream→DataStream                      | 滚动计算，每次传入的是当前元素和上一个元素<br />keyedStream.reduce(new ReduceFunction<Integer>() {     <br />  @Override     <br />  public Integer reduce(Integer value1, Integer value2)   throws Exception {         <br />    return value1 + value2;     <br />  } }); |
| **Fold** <br />KeyedStream→DataStream                        | 滚动计算，每次传入的是当前元素和上一个元素.  <br />例如sequence (1,2,3,4,5), 输出 "start-1", "start-1-2", "start-1-2-3", ...<br />DataStream<String> result =   <br />keyedStream.fold("start", new FoldFunction<Integer, String>() {     <br />  @Override     <br />  public String fold(String current, Integer value) {        <br />     return current + "-" + value;     <br />  }   }); |
| **Aggregations** KeyedStream→DataStream                      | keyedstream的一些滚动计算函数。 min和minBy的区别是，min只返回最小值，minBy返回最小值那一整条记录.<br />keyedStream.sum(0); <br />keyedStream.sum("key"); <br />\|keyedStream.min(0); <br />\|keyedStream.min("key"); <br />keyedStream.max(0); <br />keyedStream.max("key"); <br />keyedStream.minBy(0); <br />keyedStream.minBy("key"); <br />keyedStream.maxBy(0); <br />keyedStream.maxBy("key"); |
| **Window** <br />KeyedStream→WindowedStream                  | 窗口函数，Windows对KeyedStreams起作用。 dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data    ` |
| **WindowAll** <br />DataStream→AllWindowedStream             | 窗口函数，作用同window。windowAll会把所有的event放到一个task中。<br />`dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5)));   ` |
| **Window Apply** <br />WindowedStream → DataStream AllWindowedStream→DataStream | apply方法给window函数添加用户自定义的逻辑，如果是WindowAll，则需实现AllWindowFunction。<br /><img src="./image-20200703145256874.png" alt="image-20200703145256874" style="zoom:100%;" /> |
| **Window Apply** <br />WindowedStream → DataStream AllWindowedStream→DataStream | apply方法给window函数添加用户自定义的逻辑，如果是WindowAll，则需实现AllWindowFunction。<br /><img src="./image-20200703145256874.png" alt="image-20200703145256874" style="zoom:100%;" /> |
| **Window Reduce** WindowedStream → DataStream                | 将reduce函数作用于窗口，返回一个reduce的值。<br />windowedStream.reduce (<br />new ReduceFunction<Tuple2<String,Integer>>() {<br />  public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {        <br />    return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);    <br />} <br />}); |
| **Window Fold**<br />WindowedStream → DataStream             | 将fold函数作用于window    `                                  |
| **Aggregations on windows** WindowedStream → DataStream      | 作用于窗口函数的函数，min和minBy的区别是，min只返回最小值，minBy返回最小值那一整条记录<br />`windowedStream.sum(0);`<br />`windowedStream.sum("key")`;<br />`windowedStream.min(0);`<br />`windowedStream.min("key");`<br />` windowedStream.max(0);<br />`<br />`windowedStream.max("key");<br />`<br />`windowedStream.minBy(0);` <br />`windowedStream.minBy("key"); `<br />`windowedStream.maxBy(0); `<br />`windowedStream.maxBy("key");    ` |
| **Union** <br />DataStream* → DataStream                     | 把两个或更多DataStream的数据union到一起，如果union自己，数据会重复。<br />`dataStream.union(otherStream1, otherStream2, ...);    ` |
| **Window Join** DataStream,DataStream → DataStream           | 根据指定的key，将两个datastream在指定的窗口中进行join操作。<br />![image-20200703150400474](./image-20200703150400474.png) |
| **Interval Join** KeyedStream,KeyedStream → DataStream       | 在指定的时间条件内，将两个keyedStream e1（left）和e2（right）用key进行join，条件e1.timestamp + lowerBound <= e2.timestamp <= e1.timestamp + upperBound && key1 == key2<br />// this will join the two streams so that // key1 == key2 && leftTs - 2 < rightTs < leftTs + 2  //upperBoundExclusive()代表是否包含上界![image-20200703150955773](./image-20200703150955773.png) |
| **Window CoGroup** DataStream,DataStream → DataStream        | window的cogroupwindow.![image-20200703151039876](./image-20200703151039876.png) |
| **Connect** <br />DataStream,DataStream→ ConnectedStreams    | “Connect”两个保留其类型的DataStream，两个DataStream类型可以不同。 连接允许两个流之间共享状态，实际上是一种join。<br />`DataStream<Integer> someStream = //... `<br />`DataStream<String> otherStream = //... `<br />`ConnectedStreams<Integer,String>connectedStreams=someStream .connect(otherStream);    ` |
| **CoMap, CoFlatMap** ConnectedStreams → DataStream           | connected data stream的map和flatMap![image-20200703151425703](./image-20200703151425703.png) |
| **Split**<br /> DataStream → SplitStream SplitStream即将弃用 | 将一个DataStream根据规则分到多个流中。![image-20200703152203435](./image-20200703152203435.png)<br />注意：放到output中的元素是value%2和else里的数，不是even和odd。even和odd只是之后取到那一批数据的名称 |
| **Select** <br />SplitStream → DataStream                    | 从splitedStream中获取一批数据.![image-20200703152401633](./image-20200703152401633.png) |
| **Iterate** <br />DataStream → IterativeStream → DataStream  | 流迭代运算，见3.4.5                                          |
| **Extract Timestamps** <br />DataStream → DataStream         | Extracts timestamps from records in order to work with windows that use event time semantics. See [Event Time](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/event_time.html).`stream.assignTimestamps (new TimeStampExtractor() {...});                ` |

Flink在延迟和吞吐量之间的把控：

默认情况下，数据会先缓存到缓存区，当大小达到阈值后发送，来增大吞吐量，但这种方式演唱了延迟。Flink提供了setBufferTimeout来控制延迟，当缓冲区大小和延迟时间达成一个条件，即发送数据。

```
env.setBufferTimeout(100);
DataStreamSource<Integer> source = env.fromElements(1, 2, 3, 4, 5, 6, 7);
source.map(new MapFunction<Integer, Integer>() {
    ...
}).setBufferTimeout(100);
```

### 3.4.5 流迭代运算

1. 创建迭代头IterativeStream
2. 定义迭代计算（map，filter等）
3. 定义关闭迭代逻辑closeWith
4. 可以定义终止迭代逻辑

```java
DataStream<Long> source = env.generateSequence(-5, 100);

IterativeStream<Long> iterate = source.iterate();
DataStream<Long> minuOne = iterate.map(new MapFunction<Long, Long>() {
    @Override
    public Long map(Long value) throws Exception {
        return (value - 1);
    }
});
DataStream<Long> stillGreaterZero = minuOne.filter(new FilterFunction<Long>() {
    @Override
    public boolean filter(Long value) throws Exception {
        return (value > 0);
    }
});
iterate.closeWith(stillGreaterZero);

stillGreaterZero.print();

env.execute("test Iterator");
```

### 3.4.6 WaterMarks

