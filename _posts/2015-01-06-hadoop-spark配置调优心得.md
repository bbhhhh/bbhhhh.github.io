---
layout:     post
title:      hadoop-2.5.0-cdh5.2.1+spark-1.2.0-bin-hadoop2.4配置调优心得
subtitle:   
date:       2015-01-06
author:     BHH
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - hadoop
    - spark
---


### 环境：

hadoop-2.5.0-cdh5.2.1  
spark-1.2.0-bin-hadoop2.4  

master,slave2  30G ram,32 vCore  
slave4         60G ram,24 vCore  
slave5         60G ram,24 vCore
 

测试用例：根据关联条件对2个文件进行关联操作，并将关联后的结果保存到HDFS上。主文件大小：13.1GB, 100086432条记录。关联文件大小：6.2GB，100086432条记录。关联后生成的文件大小：14.7GB，100086432条记录。


### Spark配置说明  
声明：以下spark配置都只需要在master节点上配置，不需要同步到slave节点上，一样生效。  
  
- Spark-env.sh

  - 先来看spark on yarn 模式  
```
#Options read in YARN client mode  
#- SPARK_EXECUTOR_INSTANCES, Number ofworkers to start (Default: 2)  
#- SPARK_EXECUTOR_CORES, Number of coresfor the workers (Default: 1).  
#- SPARK_EXECUTOR_MEMORY, Memory perWorker (e.g. 1000M, 2G) (Default: 1G)
```
  1. SPARK_EXECUTOR_INSTANCES 这个参数配在这里不起任何作用， 如果要指定executor的个数，可以通过spark-submit–num-executors 这个参数来动态指定，比修改配置文件来的方便。 
  2. SPARK_EXECUTOR_INSTANCES 这个参数的默认值是1， 配置文件中的注释是错的。 
  3. SPARK_EXECUTOR_CORES 这个参数不管是在这里还是通过 spark-submit –executor-cores参数配置，都不起作用，始终是1。 
  4. SPARK_EXECUTOR_MEMORY 这个参数在这里是起作用的，并且不单针对on yarn模式，同时针对 standalone模式。当然，也可以通过spark-submit –executor-memory 参数动态指定。 

  - 再来看standalone 模式  
```
 #Options for the daemons used in thestandalone deploy mode  
 #- SPARK_WORKER_CORES, to set the numberof cores to use on this machine    
 #- SPARK_WORKER_MEMORY, to set how muchtotal memory workers have to give executors(e.g. 1000m, 2g)  
 #- SPARK_WORKER_INSTANCES, to set thenumber of worker processes per node  
```

  1. SPARK_WORKER_CORES参数是指所有worker最多能使用的cpu 核数，默认是全部cpu资源，除非你希望限制使用数，否则不需要修改这个参数  
  2. SPARK_WORKER_MEMORY, 参数是指所有worker最多能使用的内存，默认是全部mem资源。除非你希望限制使用数，否则不需要修改这个参数  
  3. SPARK_WORKER_INSTANCES，每个节点运行的worker数量，默认是1，如果希望在standalone模式下提高每个节点同时运行的executor个数，就需要增大这个参数。  

  
  - 调优经验：  
   - 在spark standalone模式下，   
   大的SPARK_EXECUTOR_MEMORY+ 少的SPARK_WORKER_INSTANCES 运行性能要好于    
   小的SPARK_EXECUTOR_MEMORY+ 多的SPARK_WORKER_INSTANCES，在笔者硬件条件下，最佳组合是：    
   –executor-memory24G SPARK_WORKER_INSTANCES=1，该组合由于下列组合：  
   –executor-memory12G SPARK_WORKER_INSTANCES=2    
   –executor-memory6G SPARK_WORKER_INSTANCES=4  

    - 在spark on yarn模式下，  
   笔者通过测试不同的参数组合，在笔者的硬件条件下，发现最佳组合是：  
   –num-executors 32 –executor-memory 4G，该组合要优于下列组合：  
   –num-executors 16 –executor-memory 8G    
   –num-executors 8  –executor-memory 16G,   
   –num-executors 64 –executor-memory 2G   

 
- spark-defaults.conf  

该文件主要是配置log和history，内容如下：  
```
spark.eventLog.dir               hdfs://master:8020/directory  
spark.eventLog.enabled           true  
spark.yarn.historyServer.address   master:18080  
```  

奇怪的是在http://master:18080下只能看到笔者在sparkon yarn-cluster模式下运行的记录，其他spark standalone和spark on yarn-client的运行记录都看不到，不知道原因何在，若有高手知道望赐教，谢谢。

- slaves

列出spark standalone模式下所有节点的主机名（不是ip地址）。  

注意：该文件只有在sparkstandalone运行模式时才需要配置，如果是spark on yarn-client(cluster)模式，我们不需要节点的worker进程，可以将文件中的所有主机名注释掉，这样就不会在节点上启动worker进程了。如：

 #slave2  
 #slave4    
 #slave5


### Hadoop配置说明

  Hadoop的配置不同于spark，有的需要在master上配，有的需要在slave上配。  
  归纳：master上负责每个container最多能使用的资源数量。而slave上负责本节点最多能提供的资源数量。前者必须小于后者。 

- yarn-site.xml 
  - 在master节点上  
```
      <property>  
            <name>yarn.scheduler.minimum-allocation-mb</name>  
             <value>256</value>  
      </property>  
      <property>  
            <name>yarn.scheduler.maximum-allocation-mb</name>  
             <value>20480</value><!-- must less than the max mem one slave nodemanager can allocat -->  
      </property>  
      <property>  
             <name>yarn.scheduler.minimum-allocation-vcores</name>  
             <value>1</value>  
      </property>  
      <property>  
            <name>yarn.scheduler.maximum-allocation-vcores</name>  
             <value>2</value>  
      </property>  
```
 
  以上4个参数要配置在master上，配置到slave上不起作用。  
   yarn.scheduler.maximum-allocation-mb节点上最多分配给每个container的内存量，必须小于yarn.nodemanager.resource.memory-mb最小的那个节点。

  在我们的环境中，slave2的物理内存最小，它的yarn.nodemanager.resource.memory-mb= 22G，因此master上的yarn.scheduler.maximum-allocation-mb必须小于22G。

 
  该参数同时会影响到spark onyarn模式，--executor-memory 必须小于该值减去 384MB，否则spark任务报错。

  yarn.scheduler.maximum-allocation-vcores节点上分配个每个container的cpu核数，必须小于yarn.nodemanager.resource.cpu-vcores 最小的那个节点。

 
  - 在salve节点上  
   以上在master节点配的4个参数你即使配到slave上也不起作用。Slave上关键的参数是下面2个：  
```
         <property>  
                <name>yarn.nodemanager.resource.cpu-vcores</name>  
                 <value>32</value>  
                <description></description>  
         </property>  
         <property>  
                <name>yarn.nodemanager.resource.memory-mb</name>  
                <value>22528</value>  
         </property>  
```
 
  yarn.nodemanager.resource.cpu-vcores,该节点最多可使用的cpu核数（cpu资源管理），  
  yarn.nodemanager.resource.memory-mb该节点最多可使用的内存数（内存资源管理）

  这2个参数值每个节点根据节点实际硬件情况定。  

 - mapred-site.xml  
   - 在master节点上  
```
    <property>  
       <name>mapred.child.java.opts</name>  
        <value>-Xmx8192m</value>  
    </property>  
     <property>  
         <name>mapreduce.map.memory.mb</name>   
          <value>4096</value>   
     </property>  
     <property>  
         <name>mapreduce.map.cpu.vcores</name>  
          <value>1</value>  
     </property>  
     <property>  
         <name>mapreduce.reduce.memory.mb</name>  
          <value>8192</value>  
     </property>  
     <property>  
         <name>mapreduce.reduce.cpu.vcores</name>  
          <value>2</value>  
     </property>  
```

  以上几个参数配置到slave上不起作用。  
  mapred.child.java.opts配置java参数，如-Xmx，默认是200m，太小了，按照reduce任务最多需要的量来配置。但不能超过master上的yarn.scheduler.maximum-allocation-mb值。

  mapreduce.map.memory.mb每个map任务的内存量，不能超过-Xmx值。  
  mapreduce.map.cpu.vcores每个map任务的cpu核数，不能超过master上的yarn.scheduler.maximum-allocation-vcores

  mapreduce.reduce.memory.mb每个reduce任务的内存量，不能超过-Xmx值。  
  mapreduce.reduce.cpu.vcores每个reduce任务的cpu核数，不能超过master上的yarn.scheduler.maximum-allocation-vcores

 
  - 在salve节点上  
   除了不需要配置以上master的5个参数，其他与master一致即可。
