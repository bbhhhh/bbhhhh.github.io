---
layout:     post
title:      ApacheFlink学习笔记二
subtitle:   
date:       2020-4-24
author:     BHH
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
     - BigData
     - Flink
---

### 关于DataStreamAPI, Table API and SQL API ###

首先借用官方一张图片
![](https://flink.apache.org/img/api-stack.png)

上面的官方图片展示了Flink的API层次，由上之下分别是SQL/Table API， DataStream API和ProcessFunction API。封装程度由上至下越来越低，对使用者的要求也越来越高。

官方推荐使用Table/SQL API主要出于以下几点考虑：
1. 是为了屏蔽底层的流处理逻辑和分区逻辑，避免用户对底层API的不当使用引起的性能等问题；
2. Flink仍然在一个发展期，底层API的迭代更新较大，而上层的Table/SQL API相对稳定，对用户更加友好。
3. 让用户更加专注于业务逻辑而非数据流的底层处理逻辑。

虽然官方的愿望很好，但是从这段时间实际使用下来的情况看，Table/SQL API还不成熟，相当功能不完善甚至存在很多bug。具体示例如下：
```json
	JSON数据样例如下:
	{
           "actualTime": 1576654809133,
           "deviceInfo": { "deviceId": "CIOT04E86088" },
           "ponInfo": {"PONRXPower": 81}
	}
	
	"actualTime"是毫秒单位的时间戳，可以视为eventtime。
```

####  通过Table Schema API读取上述数据关键代码如下：   
```java
        StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
        bsEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
        StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings);

        ConnectorDescriptor kafkaConn = new Kafka().version("universal")
                .topic(this.topic)
                .startFromEarliest()
                .properties(kafkaProps);

        String         jsonSchema = "{\n" +
                "    \"type\": \"object\",\n" +
                "    \"properties\": {\n" +
                "        \"actualTime\": {\n" +
                "            \"type\": \"integer\"\n" +
                "        },\n" +
                "        \"deviceInfo\": {\n" +
                "            \"type\": \"object\",\n" +
                "            \"properties\": {\n" +
                "                \"deviceId\": {\n" +
                "                    \"type\": \"string\"\n" +
                "                }\n" +
                "            }\n" +
                "        },\n" +
                "        \"ponInfo\": {\n" +
                "            \"type\": \"object\",\n" +
                "            \"properties\": {\n" +
                "                \"PONRXPower\": {\n" +
                "                    \"type\": \"integer\"\n" +
                "                }\n" +
                "            }\n" +
                "        }\n" +
                "    }\n" +
                "}";
        
        FormatDescriptor jsonFormat = new Json().failOnMissingField(false)
                .jsonSchema(jsonSchema);

        Schema tableSchema = new Schema()
                .field("actualTime", DataTypes.TIMESTAMP(3))
                .rowtime(new Rowtime().timestampsFromField("actualTime").watermarksPeriodicAscending())
                .field("deviceInfo", DataTypes.ROW(DataTypes.FIELD("deviceId", DataTypes.STRING())))
                .field("ponInfo", DataTypes.ROW(DataTypes.FIELD("PONRXPower", DataTypes.BIGINT())));

        bsTableEnv.connect(kafkaConn)
                .withFormat(jsonFormat)
                .withSchema(tableSchema)
                .inAppendMode()
                .createTemporaryTable("rxpower_detail");
```
	
运行上述代码，会得到以下Exception：  
```java
Exception in thread "main" org.apache.flink.table.api.TableException: Window aggregate can only be defined over a time attribute column, but TIMESTAMP(3) encountered.
	at org.apache.flink.table.planner.plan.rules.logical.StreamLogicalWindowAggregateRule.getInAggregateGroupExpression(StreamLogicalWindowAggregateRule.scala:51)
	at org.apache.flink.table.planner.plan.rules.logical.LogicalWindowAggregateRuleBase.onMatch(LogicalWindowAggregateRuleBase.scala:79)

```

问题出在这段代码`.rowtime(new Rowtime().timestampsFromField("actualTime").watermarksPeriodicAscending())`，它的作用就相当于DataStream API中的`assignTimestampsAndWatermarks`，但实际没起作用，Flink认为字段'actualTime'不具备TimeAttribute，因此无法进行Window操作，导致异常。具体bug描述可以参考:[https://issues.apache.org/jira/browse/FLINK-16160](https://issues.apache.org/jira/browse/FLINK-16160)，看bug中的具体描述，不仅是`rowtime()`不起作用，`proctime()`也不起作用，**这就非常致命了，基本宣告了通过Table Schema API创建Table这条路被封死了。**

上面的代码还有一个不确定的地方是`new Schema().field("actualTime", DataTypes.TIMESTAMP(3))`，Flink能否将long型的毫秒数转成TIMESTAMP类型不得而知。这里定义的类型是`TIMESTAMP(3)`,而不是该字段本来的`BIGINT()`或`DECIMAL()`型，主要是因为FLink只能对`TIMESTAMP`类型字段进行window操作，如果定义成数字型，Flink会抛异常。

####  通过Table DDL API读取上述数据关键代码如下：
* 如果DDL语句这样写：   
```java
        String createTable = "CREATE TABLE rxpower_detail (\n" +
                "  actualTime TIMESTAMP(3),\n"
                + " deviceInfo ROW(deviceId string), \n" +
                " ponInfo ROW(PONRXPower DECIMAL(38,18)), \n"
                + " WATERMARK FOR actualTime AS actualTime - INTERVAL '5' SECOND \n" +
                ") ";	
```
运行后会得到以下Exception：
```java
    Exception in thread "main" org.apache.flink.table.api.ValidationException: Type TIMESTAMP(3) *ROWTIME* of table field 'actualTime' does not match with the physical type LEGACY('DECIMAL', 'DECIMAL') of the 'actualTime' field of the TableSource return type.
	at org.apache.flink.table.utils.TypeMappingUtils.lambda$checkPhysicalLogicalTypeCompatible$4(TypeMappingUtils.java:164)
```   
是说DDL定义的类型与原始JSON类型不匹配。

* 但如果DDL语句这样写：   
```java
        String createTable = "CREATE TABLE rxpower_detail (\n" +
                "  actualTime DECIMAL(38,18),\n"
                + " deviceInfo ROW(deviceId string), \n" +
                " ponInfo ROW(PONRXPower DECIMAL(38,18)), \n"
                + " WATERMARK FOR actualTime AS actualTime - INTERVAL '5' SECOND \n" +
                ") ";	
```
运行后又会得到以下Exception：      
```java
 org.apache.calcite.runtime.CalciteContextException: From line 5, column 30 to line 5, column 61: Cannot apply '-' to arguments of type '<DECIMAL(38, 18)> - <INTERVAL SECOND>'. Supported form(s): '<NUMERIC> - <NUMERIC>'
'<DATETIME_INTERVAL> - <DATETIME_INTERVAL>'
'<DATETIME> - <DATETIME_INTERVAL>'
```
是说不支持对`DECIMAL`类型进行时间操作。

* 无奈DDL语句只能这样写：    
```java
        String createTable = "CREATE TABLE rxpower_detail (\n" +
                "  actualTime DECIMAL(38,18),\n"
                + " rowtime as TO_TIMESTAMP(FROM_UNIXTIME(cast(actualTime as INTEGER))) ,\n"
                + " deviceInfo ROW(deviceId string), \n" +
                " ponInfo ROW(PONRXPower DECIMAL(38,18)), \n"
                + " WATERMARK FOR rowtime AS rowtime - INTERVAL '5' SECOND \n" +
                ") ";	
```
额外的增加一个rowtime 字段，值从actualTime字段通过函数转换成`TIMESTAMP`类型，然后在rowtime这个字段上进行window操作，虽然时间字段的问题解决了，但是运行后又出现以下Exception：     
```java
  org.apache.flink.table.api.ValidationException: Type ROW<`PONRXPower` DECIMAL(38, 18)> of table field 'ponInfo' does not match with the physical type ROW<`PONRXPower` LEGACY('DECIMAL', 'DECIMAL')> of the 'ponInfo' field of the TableSource return type.
	at org.apache.flink.table.utils.TypeMappingUtils.lambda$checkPhysicalLogicalTypeCompatible$4(TypeMappingUtils.java:164)
	at org.apache.flink.table.utils.TypeMappingUtils$1.defaultMethod(TypeMappingUtils.java:277)
	at org.apache.flink.table.utils.TypeMappingUtils$1.defaultMethod(TypeMappingUtils.java:254)
```
是说嵌套在`ROW`类型中的`DECIMAL(38, 18)`类型与原始的 `LEGACY('DECIMAL', 'DECIMAL')`类型不匹配，彻底无解了，网上搜了一遍，发现竟然还是一个bug,具体请参考：[https://issues.apache.org/jira/browse/FLINK-16800](https://issues.apache.org/jira/browse/FLINK-16800)。**也就是说，如果原始JSON的格式超过一层，那么基本上宣告DDL这种方式创建Table也不可行。**


***
通过上述对Table API的使用对比，发现Flink的上层API问题还很多，主要是类型转换，时间属性等关键点，距离稳定商用还差很远。

结论：
如果项目一定要选择Flink，建议使用相对稳定的DataStream API。


EOF




