---
layout: post
title: "Log4j与Logback导致程序在Linux中无法输出日志"
date: 2013-11-02 21:37
comments: true
categories: 
---

前几天，在某个服务器上，部署一个新的工程，但是发现无日志文件输出。

第一时间想到的是logback的路径问题，工程中的logback.xml的的LOG_HOME为../logs。将之替换为/home/logs后仍无文件。

冥思苦想后，怀疑是linux句柄数导致的问题，通过ulimit命令发现句柄数为1024，随后通过lsof命令查看已有的进程数，发现已经大于1024。修改为句柄的最大值，并重新启动工程，发现日志也没有输出，再用lsoft也查不到工程有对应的文件句柄。

在漫无头绪的情况下，决定休息一个晚上，第二天在定位。


决定还是从路径抓起，在项目打印一些特殊信息，来观察一下logback的相关配置信息。在相关的启动类中，加入了如下代码：
```java
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
System.out.println(lc);
```
在本机的windows环境中，此代码通过。上传到服务器并启动，控制台输出了如下日志：
```java
Exception in thread "main" java.lang.ClassCastException: org.slf4j.impl.Log4jLoggerFactory cannot be cast to ch.qos.logback.classic.LoggerContext

```
这是我想到，此工程用的是logback作为日志输出工具，为什么会有log4j？


与此同时，控制台也出现如下几行日志：
```java
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/battle/lib/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/battle/lib/logback-classic-1.0.13.jar!/org/slf4j/impl/StaticLoggerBinder.class]

```

为此怀疑是log4j与logback并存的问题。

立即修改pom.xml，增加log4j-over-slf4j.jar，删除log4j的配置信息。重新部署工程后，日志文件就就出现了。

究其原因，是log4j与logback并存导致无日志文件输出。在服务器上，程序优先加载log4j，但是没有为之包含相应的配置文件，导致日志文件没有创建。

经过此战，也收获不少心得：

* 关注告警信息，它是问题的突破口
* 一个工程内，尽量只使用一种日志打印框架




