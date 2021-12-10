---
layout:     post
title:      ApacheMaven使用技巧
subtitle:   
date:       2021-12-10
author:     BHH
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
     - Java
     - Maven
---

### ApacheMaven使用技巧 ###


#### 1. 如何让Maven中maven-antrun-plugin插件走代理下载依赖包

许多大型项目的pom文件中需要使用maven-antrun-plugin插件下载软件包，比如Apache atlas项目，但是虽然在maven的settings.xml中配置了代理，但是对maven-antrun-plugin无效，可以通过在mvn命令中显示的增加代理配置使maven-antrun-plugin走代理。实例如下：


```Shell
mvn clean -DskipTests -Dhttp.proxyHost=<ip> -Dhttp.proxyPort=<port> -Dhttps.proxyHost=<ip> -Dhttps.proxyPort=<port> package -Pdist,embedded-hbase-solr
```



EOF



