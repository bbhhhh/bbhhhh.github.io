---
layout:     post
title:      ApacheFlink学习笔记一
subtitle:   
date:       2020-4-21
author:     BHH
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
     - BigData
     - Flink
---

作为流处理框架的新秀，这两年ApachFlink非常热，所以最近花了些时间学习了一下，一些基本概念Apache官方文档已经比较全面不再复述，这里主要将实际学习测试中遇到的几个知识点整理出来供学习参考。

## 关于Event Time,Processing Time,WaterMark和Window

  - Event Time是事件本身实际发生的时间。
  - Processing Time是实际处理某个Event的时间，等同于系统当前时间。
  - WaterMark（水位线）, 只有当根据Event Time进行Window的时候`env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)`才需要设置watermark。Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Apache Flink Source或者自定义的Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。 Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。watermark默认设置间隔是200ms。
	  - WaterMark的值可以基于EventTime计算，也可以基于ProcessingTime（系统时间）计算。
	  - 基于EventTime计算WaterMark一般是为了克服Event乱序到达的问题。但是由于watermark time的生成完全依赖于event time，因此当stream中断或结束后，由于没有更大的event time到达，因此watermark也不会更新，从而可能无法触发最后一批收到的event所属的window的关闭操作，也就意味着最后这个window的计算任务始终无法执行。
	  - 基于ProcessingTime计算WaterMark，无论是否有后续event到达，ProcessingTime始终在走，所以WaterMark会保持更新，无论是否有后续event到达，window肯定会被fire并开始计算。
  - Window关闭触发条件
	  - watermark时间 >= 窗口的结束时间
		  - 	如果基于ProcessingTime且以5s间隔设置window，那么当系统时间到达0s，5s，10s，。。。时就满足本条件。
		  - 	如果基于EventTime且以5s间隔设置window，此时必须设置watermark算法。只有当大于等于window endTime的watermark出现后，Flink才会认为该window的所有event已经到达，关闭window并开始计算。没有到达的event将会被抛弃或进行旁路处理,可以参考[Getting late data as a side output](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/operators/windows.html#getting-late-data-as-a-side-output)。
	  - 在窗口的时间范围(左闭右开)内有数据


下面上代码以便更深入的理解这些概念：
 
```
public static final DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

public static final Time MAX_OUT_OF_ORDERNESS = Time.seconds(3); // EventTime延迟3s作为watermark
public static final Time MAX_WATERMARK_DELAY = Time.seconds(3);  // 系统时间延迟3s作为watermark 
public static final Time TIME_WINDOW_SIZE = Time.seconds(5); // 5s间隔的window

public static void main(String... args) {
Properties kafkaProps = (Properties) KafkaConfig.getInstance().getKafkaProps().clone();
kafkaProps.put(ConsumerConfig.GROUP_ID_CONFIG, this.getClass().getSimpleName());

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);  // 设置Stream处理的Time属性，默认是ProcessingTime

   // stream source event format: JSON Node
   // mapped stream event format: Tuple4<time in millsecs, time in readable string, deviceid, number>
   SingleOutputStreamOperator<Tuple4<Long, String, String, Integer>dataStream = env
                                      .addSource(new FlinkKafkaConsumer<>(this.topic, new JsonNodeDeserializationSchema(), kafkaProps))
                                     .map(new MyMapFunc()) // transform JSON node to Tuple4<>

                                   // 根据Event Time计算watermark
                                   //.assignTimestampsAndWatermarks(new MyBoundedOutOfOrdernessTimestampExtractor(MAX_OUT_OF_ORDERNESS))

                                  // 根据processing time (系统时间）计算watermark
                                  .assignTimestampsAndWatermarks(new MyAssignerWithPeriodicWatermarks())

                                 .keyBy(2) // key by deviceid
                                 .timeWindow(TIME_WINDOW_SIZE)  // 5s间隔设置windown
                                .reduce(new MyReduceFunc())  

       dataStream.print();

      try {
            env.execute("print low optical power device event.");
      } catch (Exception e) {
            logger.error(e.getMessage(), e);
     }
  }
``` 


上述代码大致完成的功能包括：
   
  1. 从Kafka读入符合JSON格式的stream。
  2. 将JSONNode型map成Tuple4<>型，便于后期处理。
  3. 自定义实现EventTime抽取及WaterMark设置。
  4. 设置Tumbling Window (关于Flink的三种Window类型的介绍可以自行参考Flink官方文档)
  5. 以DeviceId为Key进行group
  6. 对每个window内的每个deviceid进行reduce计算。
 
根据ProcessingTime处理Stream相对简单，这里我们着重讨论下根据EventTime处理Stream时的情况，这是绝大多数业务场景需要解决的问题。

  首先，当 `env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);` 时，表示Flink按照EventTime进行stream处理，此时必须调用 `assignTimestampsAndWatermarks（）`，用来分配一个EventTime抽取方法和WaterMark计算方法。
  
 #### 实现一：周期性的根据系统时间计算watermark的内部类:
     
```
private static class MyAssignerWithPeriodicWatermarks implements AssignerWithPeriodicWatermarks<Tuple4<Long, String, String, Integer>> {
  private static final long serialVersionUID = 1L;

    @Override
   /**
   *  Tuple4<>中的第1个元素表示event time
   */
  public long extractTimestamp(Tuple4<Long, String, String, Integer> element, long previousElementTimestamp) {
    return element.f0;
  }

    /**
    * watermark time的生成依赖于系统时间，因此watermark time始终会保持更新并触发window关闭，因此即使stream中断或关闭，
    * 最后收到的event只要在window关闭前到达，就会被计算在内。
   */
   @Override
   public Watermark getCurrentWatermark() {
        return new Watermark(System.currentTimeMillis() - MAX_WATERMARK_DELAY.toMilliseconds());
   }

}
```

   该内部类中第二个方法根据系统当前时间延迟3s计算watermark时间。实现一举例有如下event stream：
     
  ![](https://bbhhhh.github.io/img/flink-20200424111935.png)

  * T(n)代表系统时间  
  * e(n)代表eventtime为n的event事件流，并假定e(4)事件延迟严重，实际是在T(9)时到达的。  
  * 按照5s间隔设置TumblingWindow，那么每个window的结束时间为w(5), w(10), w(15)...。  
  * w[0-5)窗口应该包含e(1),e(3),e(4)  
  * w[5-10)窗口应该包含e(6)  
  * w[10-15)窗口应该包含e(12),e(14)  
  根据上面的watermark算法  
  1. 当系统时间走到T(8)时，watermark为wt(5)，大于等于w(5)，w[0-5)窗口关闭条件满足，此时虽然e(4)尚未到达，但仍然会关闭window并对 e(1),e(3)开始计算。  
  2. 此时收到延迟到达的e(4)，由于w[0-5)已经关闭，该事件将被抛弃。  
  3. 当系统时间走到T(13)时，watermark为wt(10)，大于等于w(10)，w[5-10)窗口关闭条件满足，w(10)关闭并对 e(6)开始计算。  
  4. 假设e(14)以后流中断或结束了，但系统时间始终在走，当走到T(18)时， watermark为wt(15)，大于等于w(15)，w[10-15)窗口关闭条件满足，w(15)关闭并对e(12), e(14)开始计算。

 
  #### 实现二：根据EventTime计算watermark的内部类
   
```
    private static class MyBoundedOutOfOrdernessTimestampExtractor
            extends BoundedOutOfOrdernessTimestampExtractor<Tuple4<Long, String, String, Integer>> {

        public MyBoundedOutOfOrdernessTimestampExtractor(Time maxOutOfOrderness) {
            super(maxOutOfOrderness);
        }

        /**
         *extract event time from Tuple<eventtime in long, eventtime in string, deviceid,count>.f0 
         */
        @Override
        public long extractTimestamp(Tuple4<Long, String, String, Integer> element) {
            return element.f0;
        }

    }
```
   该内部类继承了Flink内置的BoundedOutOfOrdernessTimestampExtractor类，该父类提供了一个用于防止Event乱序到达的watermark默认实现。子类必须实现extractTimestamp()方法。默认的watermark算法是根据收到的Event的最大EventTime延迟maxOutOfOrderness秒计算watermark。本例中maxOutOfOrderness设为3s。实现二举例有如下event stream：

![](https://bbhhhh.github.io/img/flink-20200424112016.png)
  
 * T(n)代表系统时间
 * e(n)代表eventtime为n的event事件流，并假定e(4)事件延迟严重，实际是在T(9)时到达的。
 * 按照5s间隔设置TumblingWindow，那么每个window的结束时间为w(5), w(10), w(15)...。
 * w[0-5)窗口应该包含e(1),e(3),e(4)
 * w[5-10)窗口应该包含e(6)
 * w[10-15)窗口应该包含e(12),e(14)
     根据上面的watermark算法
1. 当收到e(3)，watermark为wt(0)，收到e(6)，watermark为wt(3)。
2. 收到e(4)后，根据算法，watermark保持不变，仍旧为wt(3)，由于w[0-5)尚未关闭，可以加入该窗口。
3. 收到e(12)，watermark为wt(9)， 大于等于w(5)，w[0-5)窗口关闭条件满足，对e(1),e(3),e(4)进行计算。
4. 收到e(14)，watermark为wt(11)， 大于等于w(10)，w[5-10)窗口关闭条件满足，对e(6)进行计算。
5. 由于e(14)以后流中断了，再没有新的event到达，因此watermark不会更新，始终停留在wt(11)，从而窗口w[10-15)始终无法满足关闭条件，对e(12),e(14)的计算也就始终无法开始，除非收到新的event事件。


*虽然实现二能解决乱序Event问题，但是这个取决于maxOutOfOrderness阈值的大小，根据上图，如果该值设为1s的话，当收到e(6)时，watermark将会更新为wt(5)，满足窗口w[0-5)关闭条件， e(6)事件仍然会被丢弃。所以该阈值的大小设置取决于经验值，但又不能设置的过大以防止窗口等待时间过长数据积压严重导致处理性能下降。*




EOF




