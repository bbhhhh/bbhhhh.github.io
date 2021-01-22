---
layout:     post
title:      KeepAlive参数为On时不同Apache版本的表现行为差异对比
subtitle:   
date:       2020-05-06
author:     BHH
header-img: img/post-bg-e2e-ux.jpg
catalog: true
tags:
    - HTTP, Apache
---

### KeepAlive参数为On时不同Apache版本的表现行为差异对比



最近项目碰到一个奇怪的问题，Apache httpd.conf配置文件中的参数`KeepAlive On`在不同版本中表现出的行为不一致，导致现网应用出现故障。特记录下来供分析研究。



#### 环境一：Server version: Apache/2.2.14 (Unix)，应用正常 ####

Http post:

```http
POST /ACS-server/BasicACS HTTP/1.1
Content-type: text/xml; charset=UTF-8
Content-Length: 4029
Host: 135.251.208.152:9090
Connection: Keep-Alive
```

401 Response：

```http
HTTP/1.1 401 Authorization Required
Date: Wed, 06 May 2020 05:54:59 GMT
Server: Apache
Content-Length: 1468
WWW-Authenticate: Basic realm="default"
X-Powered-By: Servlet/2.5 JSP/2.1
Keep-Alive: timeout=15, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

WireShark 抓包:

![](https://bbhhhh.github.io/img/apache2.2.14-KeepAliveOn.png)

从WireShark抓包看到，Server在收到客户端要求`Connection: Keep-Alive`请求后也回复了`Connection: Keep-Alive`, 回复了401后并没有关闭TCP连接，后续的HTTP请求仍然复用了该TCP连接。该环境下，所有终端都能够完成HTTP认证session，应用表现正常。



#### 环境二：Server version: Apache/2.4.12 (Unix)，部分终端应用不正常 ####

Http post:

```http
POST /cwmpWeb/WGCPEMgt HTTP/1.1
Content-type: text/xml; charset=UTF-8
Content-Length: 4029
Host: 135.251.218.88:9090
Connection: Keep-Alive
```

401 Response：

```http
HTTP/1.1 401 Unauthorized
Date: Wed, 06 May 2020 05:46:37 GMT
Server: Apache
WWW-Authenticate: Basic realm="default"
Connection: close
Content-Length: 381
Content-Type: text/html; charset=iso-8859-1
```

WireShark 抓包:

![apache2.4.12-KeepAliveOn](https://bbhhhh.github.io/img/apache2.4.12-KeepAliveOn.png)

从WireShark抓包看到，即使客户端要求`Connection: Keep-Alive`，Server仍然回复了`Connection: close`，且关闭了TCP连接，后续的HTTP请求需要重新建立TCP连接完成。**问题就出在这里，部分终端支持跨TCP连接完成HTTP认证Session，部分终端不支持，它们必须在同一个TCP连接中完成HTTP认证Session，因此在环境二中，部分终端的应用表现正常，部分终端异常，引发平台故障。**