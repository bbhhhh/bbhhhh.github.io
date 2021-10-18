---
layout:     post
title:      ApacheFlink学习笔记三
subtitle:   
date:       2021-10-10
author:     BHH
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
     - BigData
     - Flink
---

### 使用Flink1.14.0 TableAPI对EventTime进行Window操作碰到的问题 ###

最近有空又开始继续研究Flink了，直接上最新的稳定版1.14.0，没想到出师不利，在使用TableAPI进行Window聚合操作时碰到以下问题：
如果用EventTime进行Window操作，转换成DataStream后调用print()方法控制到没有任何输出；但改成ProcessingTime进行Window操作却一切正常。

使用DataStream API对EventTime进行Window操作也是正常的，百思不得其解。

核心pom如下：
```xml
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>1.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_2.12</artifactId>
            <version>1.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_2.12</artifactId>
            <version>1.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_2.12</artifactId>
            <version>1.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-json</artifactId>
            <version>1.14.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-api-java-bridge_2.12</artifactId>
            <version>1.14.0</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner_2.12</artifactId>
            <version>1.14.0</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.12</artifactId>
            <version>1.14.0</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-common</artifactId>
            <version>1.14.0</version>
<!--            <scope>provided</scope>-->
        </dependency>
```
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

public class CountEventsPerDeviceByTumbleWindowTableAPI extends FlinkConsumer {
    private static final Logger logger = LoggerFactory.getLogger(CountEventsPerDeviceByTumbleWindowTableAPI.class);

    /**
    * @param topic
    * @throws
    */
    public CountEventsPerDeviceByTumbleWindowTableAPI(String topic) {
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


        String createTable = "\n CREATE TABLE rxpower_detail (\n" +
                "    actualTime BIGINT,\n" +
                "    ponInfo ROW(PONRXPower INT),\n" +
                "    deviceInfo ROW(deviceId STRING), \n" +
                "    event_time AS TO_TIMESTAMP(FROM_UNIXTIME(actualTime/1000,'yyyy-MM-dd HH:mm:ss')) ,\n" +
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
                + "\n select window_start as w_s,window_end as w_e, window_time as w_t, deviceId, count(deviceId) as lc "
                + "\n from TABLE (TUMBLE(TABLE rxpower_detail,DESCRIPTOR(event_time), INTERVAL '5' SECOND))"
//                + " from TABLE (TUMBLE(TABLE rxpower_detail,DESCRIPTOR(proc_time), INTERVAL '5' SECOND))"
                + "\n group by window_start,window_end,window_time, deviceId  "
                +"\n"
                ;
        logger.info(selectSql);
        tableEnv.executeSql(selectSql);
//
        selectSql = "select * from tumble_windowed_result " ;

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
        new CountEventsPerDeviceByTumbleWindowTableAPI("topic.periodic").run();
    }
}
```

运行上述代码，模拟向Kafka发送测试数据，控制台没有任何输出，但如果将上面的
```sql
from TABLE (TUMBLE(TABLE rxpower_detail,DESCRIPTOR(event_time), INTERVAL '5' SECOND))
```
换成Processing Time字段，改成
```sql
from TABLE (TUMBLE(TABLE rxpower_detail,DESCRIPTOR(proc_time), INTERVAL '5' SECOND))
```
就一切正常了，控制台能正常输出统计数据。

另外，如果将Flink版本换成1.13.2，并使用Blink table planner，不管是Event Time还是ProcessingTime又都是正常的。换成1.13.2后的核心pom如下：
```xml
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_2.12</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_2.12</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_2.12</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-json</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-api-java-bridge_2.12</artifactId>
            <version>1.13.2</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner_2.12</artifactId>
            <version>1.13.2</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.12</artifactId>
            <version>1.13.2</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner-blink_2.12</artifactId>
            <version>1.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-common</artifactId>
            <version>1.13.2</version>
<!--            <scope>provided</scope>-->
        </dependency>
```

这个问题也不知是不是Flink1.14.0版本的问题。

EOF



