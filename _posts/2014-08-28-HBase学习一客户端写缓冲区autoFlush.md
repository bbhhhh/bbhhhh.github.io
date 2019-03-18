---
layout:     post
title:      HBase 学习一 客户端写缓冲区 autoFlush
subtitle:   
date:       2014-08-28
author:     BHH
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - HBase
---


HBase的表操作，默认情况下客户端写缓冲区是关闭的，即table.isAutoFlush() = true， 这种情况下，对表的单行操作会实时发送到服务端完成。

因此，对于海量数据插入，修改，RPC通信频繁，效率比较低。这种场景下，可以通过激活客户端缓冲区，批量提交操作请求，提高操作效率。



下面是一个简单的关于autoFlush的测试代码：
```
public static void autoFlushTest(){
		HTable table;
		
		try {
			table = new HTable(conf, "test3");
			
			//默认是不激活，实时提交服务端处理请求.
			System.out.println("=====auto flush:" + table.isAutoFlush());  
			
			if (table.isAutoFlush())
				table.setAutoFlush(false);    //激活缓冲区
			
			System.out.println("=====auto flush2:" + table.isAutoFlush());
			
			
			Put put = new Put(Bytes.toBytes("r1"));
			
			put.add(Bytes.toBytes("colf2"), Bytes.toBytes("q1"), Bytes.toBytes("v3"));
			put.add(Bytes.toBytes("colf2"), Bytes.toBytes("q2"), Bytes.toBytes("v4"));
			
			table.put(put);
			
			// 当setAutoFlush=false时，只有当缓冲区满或调用table.close()时
			// 或调用table.flushCommits()时
			//客户端才会将table操作提交服务端执行。
			
			//table.flushCommits();
			table.close();
			
			System.out.println("=====auto flush4:" + table.isAutoFlush());
			
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
	}
```