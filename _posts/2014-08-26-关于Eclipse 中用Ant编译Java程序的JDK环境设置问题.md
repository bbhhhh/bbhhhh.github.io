---
layout:     post
title:      关于Eclipse 中用Ant编译Java程序的JDK环境设置问题
subtitle:   
date:       2014-08-26
author:     BHH
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - Eclipse
    - java
    - ant
---

日前在开发项目过程中碰到一个Java编译环境配置问题，折腾了不少时间，特写下来以备后用：


问题是这样的，有一个java程序，通过Eclipse 的export jar功能能够正常编译并打包，但用Ant编译却报下面的错误：  
`"java.lang.UnsupportedClassVersionError: com/sun/tools/javac/Main : Unsupported major.minor version 51.0"`

本机上装了2个JDK，第一个装的是JDK1.7  
当时Eclipse中的环境设置是这样的：  
Project默认JRE环境：C:\Program Files\Java\jre6 （项目需要1.6环境）

Ant环境：Window->Preferences->Ant->Runtime->Global Entries配的是“C:\Program Files\Java\jdk1.7.0_67\lib\tools.jar"

最终分析发现正是由于以上两项配置互相冲突，导致Ant编译失败。

**解决办法一：**

将Ant Global Entries改为：“C:\Program Files\Java\jdk1.6.0_45\lib\tools.jar"


**解决办法二：**

将Project JRE环境指向JDK根目录而不是JRE根目录："C:\Program Files\Java\jdk1.6.0_45"

此时Ant Global Entries配不配都不受影响。


比较彻底的方案是办法二，如果指向的是JRE根目录而不是JDK根目录，并且Ant Global Entries没有配置的话，会导致无法找到javac 命令的错误。

希望给碰到同样问题的同学有所帮助。

EOF