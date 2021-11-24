---
layout:     post
title:      ApacheFlink学习笔记四
subtitle:   
date:       2021-10-18
author:     BHH
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
     - BigData
     - Flink
---

### Flink TableAPI Over Aggregation操作碰到的问题 ###

###### !!! updated by 2021.11.24
###### 证实了，本文描述的时间类型问题确实是Flink的bug，在1.13.3和1.14.0中已经解决了，特地修改一下。
###### !!!

最近在学习Flink TableAPI Over聚合操作时又碰到了奇怪的问题，在Flink1.13.2版本上，当Order By字段是TIMESTAMP_LTZ类型时，会抛错；但如果是TIMESTAMP类型时就是正常的。

测试代码如下：

```java
package com.nokia.itms.flink.sql;

import java.util.Properties;

import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.nokia.itms.flink.FlinkConsumer;
import com.nokia.itms.kafka.KafkaConfig;

public class PrintLowPowerDevicesByOverAggregationTableAPI extends FlinkConsumer {
    private static final Logger logger = LoggerFactory.getLogger(PrintLowPowerDevicesByOverAggregationTableAPI.class);

    /** 
    * @param topic
    * @throws
    */
    public PrintLowPowerDevicesByOverAggregationTableAPI(String topic) {
        super(topic);

    }

    @Override
    public void run() {
        Properties kafkaProps = (Properties) KafkaConfig.getInstance().getKafkaProps().clone();
        kafkaProps.put(ConsumerConfig.GROUP_ID_CONFIG, this.getClass().getSimpleName());
        logger.info("Event max out of orderness = {} {}",MAX_OUT_OF_ORDERNESS.getSize(),MAX_OUT_OF_ORDERNESS.getUnit());

        StreamExecutionEnvironment execEnv = StreamExecutionEnvironment.getExecutionEnvironment();

        EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().inStreamingMode().build();
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(execEnv, bsSettings);
        logger.info("before local tz={}",tableEnv.getConfig().getLocalTimeZone().toString());


        String createTable = "\n CREATE TABLE rxpower_detail (\n" +
                "    actualTime BIGINT,\n" +
                "    ponInfo ROW(PONRXPower INT),\n" +
                "    deviceInfo ROW(deviceId STRING), \n" +
                "    event_time AS TO_TIMESTAMP(FROM_UNIXTIME(actualTime/1000)) ,\n" +
                // !!! the event time attribute which produced by
                // below TO_TIMESTAMP_LTZ function can't be used in OVER Aggregation, will throw
                // 'OVER windows' ordering in stream mode must be defined on a time attribute' Exception, it's maybe a bug.
                // !!!
//                "    event_time AS TO_TIMESTAMP_LTZ(actualTime,3) ,\n" +
                "    WATERMARK FOR event_time AS event_time - INTERVAL '" +
                  MAX_OUT_OF_ORDERNESS.getSize() +
                "' " +
                MAX_OUT_OF_ORDERNESS.getUnit() + " , \n" +
                " proc_time as PROCTIME() \n" +
                "    )\n" +
                " with ( \n" +
                "'connector' = 'kafka'" + ",\n" +
                "'topic' = '" + this.topic + "',\n" +
                "'properties.bootstrap.servers' = '" + kafkaProps.getProperty("bootstrap.servers")+ "',\n" +
                //"'properties.group.id' = '" + kafkaProps.getProperty("group.id")+ "',\n" +
                "'scan.startup.mode' = 'latest-offset'" + ",\n" +
                //"'scan.startup.mode' = 'group-offsets'" + ",\n" +
                "'format' = 'json'" + ",\n" +
                "'json.fail-on-missing-field' = 'false'" +",\n" +
                "'json.ignore-parse-errors' = 'true'" +"\n" +
                ")" +"\n" ;

        logger.debug(createTable);
        tableEnv.executeSql(createTable);

        String selectSql = "";

        selectSql = ""
                + "\ncreate temporary view tumble_windowed_result as "
                + "\n select window_start as w_s,window_end as w_e,window_time as w_t,deviceId, count(deviceId) as lc "
                + "\n from TABLE (TUMBLE(TABLE rxpower_detail,DESCRIPTOR(event_time), INTERVAL '5' SECOND))"
                + "\n group by window_start,window_end, window_time,deviceId "
                + "\n"
                ;

        logger.info(selectSql);

        tableEnv.executeSql(selectSql);
        selectSql = "" +
                "\n select w_s, w_e, w_t,deviceId, lc, count(deviceId) over " +
                "\n ( PARTITION BY deviceId " +
                "\n order by w_t  " +
                "\n range between INTERVAL '10' second PRECEDING AND CURRENT ROW )" +
                "\n from tumble_windowed_result " +
                "\n";
        logger.info(selectSql);


        Table table2 = tableEnv.sqlQuery(selectSql);

        DataStream<Row> resultStream = tableEnv.toDataStream(table2);


        resultStream.print();

        try {
            execEnv.execute("table api test");
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }

    }

    public static void main(String... args) {
        new PrintLowPowerDevicesByOverAggregationTableAPI("topic.periodic").run();
    }
}
```

运行上述代码，当使用
```sql
event_time AS TO_TIMESTAMP(FROM_UNIXTIME(actualTime/1000))
```
定义event_time字段时，能够正常运行并打印结果；

但如果换成
```sql
event_time AS TO_TIMESTAMP_LTZ(actualTime,3)
```
时，运行代码会抛出如下错误：
```java
Exception in thread "main" org.apache.flink.table.api.TableException: OVER windows' ordering in stream mode must be defined on a time attribute.
	at org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecOverAggregate.translateToPlanInternal(StreamExecOverAggregate.java:159)
	at org.apache.flink.table.planner.plan.nodes.exec.ExecNodeBase.translateToPlan(ExecNodeBase.java:134)
	at org.apache.flink.table.planner.plan.nodes.exec.ExecEdge.translateToPlan(ExecEdge.java:247)
	at org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecSink.translateToPlanInternal(StreamExecSink.java:104)
	at org.apache.flink.table.planner.plan.nodes.exec.ExecNodeBase.translateToPlan(ExecNodeBase.java:134)
	at org.apache.flink.table.planner.delegation.StreamPlanner.$anonfun$translateToPlan$1(StreamPlanner.scala:70)
	at scala.collection.TraversableLike.$anonfun$map$1(TraversableLike.scala:233)
	at scala.collection.Iterator.foreach(Iterator.scala:937)
	at scala.collection.Iterator.foreach$(Iterator.scala:937)
	at scala.collection.AbstractIterator.foreach(Iterator.scala:1425)
	at scala.collection.IterableLike.foreach(IterableLike.scala:70)
	at scala.collection.IterableLike.foreach$(IterableLike.scala:69)
	at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
	at scala.collection.TraversableLike.map(TraversableLike.scala:233)
	at scala.collection.TraversableLike.map$(TraversableLike.scala:226)
	at scala.collection.AbstractTraversable.map(Traversable.scala:104)
	at org.apache.flink.table.planner.delegation.StreamPlanner.translateToPlan(StreamPlanner.scala:69)
	at org.apache.flink.table.planner.delegation.PlannerBase.translate(PlannerBase.scala:165)
	at org.apache.flink.table.api.bridge.java.internal.StreamTableEnvironmentImpl.toStreamInternal(StreamTableEnvironmentImpl.java:439)
	at org.apache.flink.table.api.bridge.java.internal.StreamTableEnvironmentImpl.toStreamInternal(StreamTableEnvironmentImpl.java:434)
	at org.apache.flink.table.api.bridge.java.internal.StreamTableEnvironmentImpl.toDataStream(StreamTableEnvironmentImpl.java:358)
	at org.apache.flink.table.api.bridge.java.internal.StreamTableEnvironmentImpl.toDataStream(StreamTableEnvironmentImpl.java:331)
	at com.nokia.itms.flink.sql.PrintLowPowerDevicesByOverAggregationTableAPI.run(PrintLowPowerDevicesByOverAggregationTableAPI.java:133)
	at com.nokia.itms.flink.sql.PrintLowPowerDevicesByOverAggregationTableAPI.main(PrintLowPowerDevicesByOverAggregationTableAPI.java:147)
```
根据错误日志查看源码如下：
```java
        if (orderKeyType instanceof TimestampType
                && ((TimestampType) orderKeyType).getKind() == TimestampKind.ROWTIME) {
            rowTimeIdx = orderKey;
        } else if (orderKeyType instanceof LocalZonedTimestampType
                && ((LocalZonedTimestampType) orderKeyType).getKind() == TimestampKind.PROCTIME) {
            rowTimeIdx = -1;
        } else {
            throw new TableException(
                    "OVER windows' ordering in stream mode must be defined on a time attribute.");
        }
```
这里的逻辑判断有点晕，为什么不判断LocalZonedTimestamp的RowTime类型，直接就抛异常了呢，没办法，只能改用TO_TIMESTAMP函数。

最后，这个问题在Flink1.14.0上虽然没有报错，但是仍然碰到在笔记三中的问题，程序根本没有运行，控制台没有任何输出信息。


EOF



