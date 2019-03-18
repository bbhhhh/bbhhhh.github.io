---
layout:     post
title:      如何基于NTLM代理服务器使Maven能够联网工作
subtitle:   
date:       2014-12-18
author:     BHH
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - NTLM
    - Maven
---

最近在研究Hadoop 和Spark，需要自己编译一个spark包，用到maven工具。版本是：3.2.3，问题是公司的服务器在内网，而公司的HTTP代理是基于NTLM的，maven默认是不支持的，比如：你在settings.xml中有以下配置：
```
    <proxy>
      <id>my-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <username>domain\username</username>
      <password>password</password>
      <host>host</host>
      <port>port</port>
      <nonProxyHosts>local.net|some.host.com</nonProxyHosts>
    </proxy>
```
当执行mvn命令时，会出现无法连接NTLM代理的错误信息，



经上网查询，有网友提供解决方案如下（原帖地址：[http://www.cnblogs.com/mikelij/p/3203377.html](http://www.cnblogs.com/mikelij/p/3203377.html)）：

For the command line maven 3.1.0, we can download a extension jar file [http://repo.maven.apache.org/maven2/org/apache/maven/wagon/wagon-http-lightweight/2.2/wagon-http-lightweight-2.2.jar](http://repo.maven.apache.org/maven2/org/apache/maven/wagon/wagon-http-lightweight/2.2/wagon-http-lightweight-2.2.jar)

and save it to <command line maven home directory>/lib/ext.

按照此方案执行，已经可以连接代理了，但是代理返回了错误认证信息，
经过思考，凭直觉认为可能是maven的版本比3.1高，因此需要更高版本的jar包，于是下了一个最新版本的，
地址:[http://repo1.maven.org/maven2/org/apache/maven/wagon/wagon-http-lightweight/2.8/wagon-http-lightweight-2.8.jar](http://repo.maven.apache.org/maven2/org/apache/maven/wagon/wagon-http-lightweight/2.2/wagon-http-lightweight-2.2.jar)

替换2.2的那个包后，又提示找不到下面这个类：

org/apache/maven/wagon/shared/http/EncodingUtil.class

于是又下了一个匹配的share包，地址：[http://repo1.maven.org/maven2/org/apache/maven/wagon/wagon-http-shared/2.8/wagon-http-shared-2.8.jar](http://repo1.maven.org/maven2/org/apache/maven/wagon/wagon-http-shared/2.8/wagon-http-shared-2.8.jar)

也放到lib/ext目录下，再次运行，worked！！！



特记录下来所有步骤，以备后用。
