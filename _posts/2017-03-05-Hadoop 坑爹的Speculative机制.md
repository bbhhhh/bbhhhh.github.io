---
layout:     post
title:      Hadoop 坑爹的Speculative 机制
subtitle:   
date:       2017-03-05
author:     BHH
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Hadoop
    - HBase
---

最近一直在搞Hadoop Hbase。

我们有一个需求是从HDFS上读取输入文件，解析后输出到Hbase。

由于输入文件非常大，TB级别，为了提高写库性能，我们尝试通过map程序在所有data node上并发读取并输出到Hbase。

程序很快完成，并顺利完成入库任务。

我们写了一个统计程序用于检查导入的记录是否与输入文件中的记录数一致。

开始为了测试，输入文件并不是很大，1GB左右，2，3百万条记录，通过检查一切正常。

当换成10GB文件后，发现导入的记录数比原始文件要多，而且每次多的数量都不一样，程序没有发现任何问题，直觉告诉我应该是Hadoop的问题，幸运的是有一次测试竟然一切正常，马上打开hadoop job监控页面，比较了这次正常的job和之前错误job的监控信息，让我发现了问题所在，由于输入文件非常大，每个job都需要分配几百个map来执行导入工作，所有不正常的job中都至少有1个map被kill过，而正常的job中没有map被kill过。



于是开始研究map为什么会被kill。起初我的思维逻辑是，可能某个map执行过程中发生了异常，hadoop将该map kill并重新执行，类似于将异常map回滚。然而当我从网上了解到了hadoop 的Speculative（预测）机制后，终于真相大白了。

原来hadoop在分配完map reduce task后，会预测性的判断某个map 或reduce task所在的节点资源有限，执行会比较慢，因此他在资源更多的节点上会启动一个完全一样的map或 reduce task，同时执行，哪个先完成，就将未完成的那个task kill掉，提高整体job效率。

问题来了，由于我们是写库操作，rowkey是全局唯一的，因此两个一样的map当然会重复写同样的记录，导致最终入库的记录数比原始文件多。

小文件没有这个问题，说明map数量不够多，机器资源充足，hadoop不需要启动预测机制。



MapReduce任务有两个参数可以控制Speculative Task：  
`mapred.map.tasks.speculative.execution`： mapper阶段是否开启推测执行  
`mapred.reduce.tasks.speculative.execution`： reducer阶段是否开启推测执行

这两个参数默认都为true

hadoop2.0版本中这两个参数改为：  
`mapreduce.map.speculative`  
`mapreduce.reduce.speculative`  

java 应用可以通过如下语句关闭speculative task：  
`conf.setBoolean("mapreduce.map.speculative", false);`  
`conf.setBoolean("mapreduce.reduce.speculative", false);`

或直接修改mapred-site.xml，如下：
```xml
<property>
  <name>mapreduce.map.speculative</name>
  <value>false</value>
</property>
<property>
  <name>mapreduce.reduce.speculative</name>
  <value>false</value>
</property>
```

经过修改，再次测试100GB文件，一切正常。  

因此，当这两属性为True时，编写mapreduce程序要特别小心，尤其访问外部资源如数据库，写文件等，很容易发生重复写，产生异常。同时又额外消耗了节点资源。
