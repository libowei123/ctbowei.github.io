
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [1. Flink DataStream API](#1-flink-datastream-api)
  - [1.1. Environment](#11-environment)
    - [1.1.1. getExecutionEnvironment](#111-getexecutionenvironment)
    - [1.1.2. createLocalEnvironment](#112-createlocalenvironment)
    - [1.1.3. createRemoteEnvironment](#113-createremoteenvironment)
  - [1.2. Transform](#12-transform)
    - [1.2.1. 基本转换map/flatmap/filter](#121-基本转换mapflatmapfilter)
    - [1.2.2. keyBy](#122-keyby)
      - [1.2.2.1. reduce](#1221-reduce)
    - [1.2.3. 多流转换](#123-多流转换)
      - [1.2.3.1. split,select分流](#1231-splitselect分流)
      - [1.2.3.2. Connect,CoMap,CoFlatMap](#1232-connectcomapcoflatmap)
      - [1.2.3.3. Union](#1233-union)
    - [1.2.4. stream转换图](#124-stream转换图)
  - [1.3. 支持的数据类型](#13-支持的数据类型)
  - [1.4. User-Defined Functions](#14-user-defined-functions)
  - [1.5. Rich functions](#15-rich-functions)
  - [1.6. 数据重分区](#16-数据重分区)
- [2. Sink](#2-sink)
- [3. 案例](#3-案例)
  - [3.1. 恶意登陆检测](#31-恶意登陆检测)

<!-- /code_chunk_output -->


# 1. Flink DataStream API
## 1.1. Environment
![flinkEnvPic](/resources/flinkdatastreamapienv.png)
### 1.1.1. getExecutionEnvironment
创建一个执行环境，表示当前执行程序的上下文。如果程序是独立调用的，则此方法返回本地执行环境；如果从命令行客户端调用程序以提交到集群，则此方法返回此集群的执行环境，也就是说，getExecutionEnvironment会根据查询运行的方式决定返回什么样的运行环境，是最常用的一种创建执行环境的方式。
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.*getExecutionEnvironment*(); 
```
如果没有设置并行度，会以flink-conf.yaml中的配置为准，默认是1。
### 1.1.2. createLocalEnvironment
返回本地执行环境，需要指定并行度
```java
LocalStreamEnvironment env = StreamExecutionEnvironment.*createLocalEnvironment*(1); 
```
### 1.1.3. createRemoteEnvironment
返回集群执行环境，将jar提交到远程服务器。
需要在调用时指定JobManager的IP和端口号，并指定要在集群中运行的Jar包。
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);
```


## 1.2. Transform
### 1.2.1. 基本转换map/flatmap/filter
![map](/resources/map.jpg)
![flatmap](../resources/flatmap.jpg)
filter:return false则过滤
### 1.2.2. keyBy
![keyBy](../resources/keyby.png)
keyBy后可以聚合
- sum()
- min()
- max()
- minBy()
- maxBy()
- reduce

#### 1.2.2.1. reduce
### 1.2.3. 多流转换
#### 1.2.3.1. split,select分流
dataStream -> splitStream
根据某些特征把dataStream拆分为splitStream
splitStream仍然是一条流，需要使用select选择一条splitStream转换为dataStream
```java
input.splitStream(new OutputSelector<Sensor>()){
  @Override
  public Iterable<String> select(Sensor value){
    return (value.getTemperature()>30) ? 
    Collections.singletonList("high"): 
    Collections.singletonList("low");
  }
}
DataStream<Sensor> highStream = splitStream.select("high");
DataStream<Sensor> highStream = splitStream.select("low");
DataStream<Sensor> allStream = splitStream.select("low","high");
```

#### 1.2.3.2. Connect,CoMap,CoFlatMap
不同类型的数据流也可以连接（只能连接2条流）
dataStream -> connectedStream

#### 1.2.3.3. Union
dataStream -> dataStream
连接多条流，合并的流必须是同样的数据类型
```java
streamA.union(streamB,streamC);
```

### 1.2.4. stream转换图
![streamTransform](../resources/streamTransform.png)


## 1.3. 支持的数据类型
- java基础数据类型
- 元组 Tuples
- Java POJO
- Arrays,Lists,Maps,Enums...
  
## 1.4. User-Defined Functions
Flink暴露所有udf的接口MapFunction,FilterFunction,ProcessFunction等
例如
```java
DataStream<String> flinkTweets = tweets.filter(new FlinkFilter()); 
public static class FlinkFilter implements FilterFunction<String> { 
  @Override public boolean filter(String value) throws Exception { 
    return value.contains("flink");
  }
}
```
也可以做匿名类
```java
DataStream<String> flinkTweets = tweets.filter(
  new FilterFunction<String>() { 
    @Override public boolean filter(String value) throws Exception { 
      return value.contains("flink"); 
    }
  }
);
```
也可以传递参数
```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE "); 
DataStream<String> flinkTweets = tweets.filter(new KeyWordFilter("flink")); 
public static class KeyWordFilter implements FilterFunction<String> { 
  private String keyWord; 
  KeyWordFilter(String keyWord) { 
    this.keyWord = keyWord; 
  } 
  @Override public boolean filter(String value) throws Exception { 
    return value.contains(this.keyWord); 
  } 
}
```
## 1.5. Rich functions
每一个UDF都有对应的RichFunctions
- RichMapFunction
- RichFlatMapFunction
- RichFilterFunction
- ...
​
Rich Function有一个生命周期的概念。典型的生命周期方法有
- open()方法是rich function的初始化方法，当一个算子例如map或者filter被调用之前open()会被调用。
- close()方法是生命周期中的最后一个调用的方法，做一些清理工作。
- getRuntimeContext()方法提供了函数的RuntimeContext的一些信息，例如函数执行的并行度，任务的名字，以及state状态
```java
public static class MyMapFunction extends RichMapFunction<SensorReading, Tuple2<Integer, String>> { 
  @Override
  public Tuple2<Integer, String> map(SensorReading value) throws Exception {
    return new Tuple2<>(
      getRuntimeContext().getIndexOfThisSubtask(),
      value.getId()
      ); 
  } 
  @Override
  public void open(Configuration parameters) throws Exception { 
    System.out.println("my map open"); 
    // 以下可以做一些初始化工作，例如建立一个和HDFS的连接 
  } 
  @Override
  public void close() throws Exception { 
    System.out.println("my map close");
    // 以下做一些清理工作，例如断开和HDFS的连接 
  } 
}
```
## 1.6. 数据重分区

# 2. Sink
[参考](Connectors.md)



# 3. 案例

## 3.1. 恶意登陆检测

