---
layout: post
title: 记录碰到的一些 Java 问题(更新2014-02-26)
---

这篇文章用于记录自己碰到过的一些 Java 问题，会根据经验不断增加，以便总结。:-) 


#### CASE

线上 Tomcat 经常无故自动 Shutdown，并且非 JVM Crash 导致，因为从 Tomcat 日志中发现它 pause 到 destory 的过程，如下日志所示：
{% highlight java %}
org.apache.catalina.core.StandardServer await
2014-2-22 21:38:57 org.apache.coyote.AbstractProtocol pause
信息: Pausing ProtocolHandler ["http-bio-8110"]
2014-2-22 21:38:57 org.apache.coyote.AbstractProtocol pause
信息: Pausing ProtocolHandler ["ajp-bio-8029"]
2014-2-22 21:38:58 org.apache.catalina.core.StandardService stopInternal
信息: Stopping service Catalina 
{% endhighlight java %}


#### Solved 


从日志上来看是 Tomcat 的 hutdownhook 被被执行。后来看 Linux ssh 的退出的日志正好与 Tomcat 关闭时间存在一定的规律。也就是说 ssh 登录然后启动tomcat，再退出 ssh后， tomcat 就会自动停掉。线上服务器都是通过 tomcat-hepler.sh 脚本来启动/停止 Tomcat 实例，以下是 tomcat-helper.sh 的代码简化版本：

{% highlight bash %}
#!/bin/bash
/export/tomcatApp/xxx/bin/startup.sh
tail -f /export/logs/xxx/catalina.out 
{% endhighlight bash %}

经过分析，tomcat 启动后，当前shell进程并没有退出，而是挂住在 tail 进程，往终端输出日志内容。这种情况下，如果你直接关闭ssh终端的窗口(用鼠标或快捷键强制关闭)，则 Tomcat 进程也会退出。而如果先 ctrl-c 终止 tomcat-helper.sh 进程，然后再关闭 ssh 终端的话，则 Tomcat 进程不会退出。 

解决此办法，就是在启动 Tomcat 的时候增加 nohup 的支持：

{% highlight bash %}
nohup $Base/$NAME/bin/startup.sh >/dev/null 2>&1
{% endhighlight bash %}

关于该问题的具体原因更深入的分析可参考下面的链接：

#### References
* [tomcat进程意外退出的问题分析](http://hongjiang.info/why-kill-2-cannot-stop-tomcat/)


<hr/>


#### CASE

某 Java Web API 程序，在进行 LoadRunner 压力测试时 200/user 发现8个 CPU(4个双核) used 基本上都在 90%~100% 之间了，HTTP 平均响应时间 2/s 左右，TPS 极其低。 


#### Solved 

既然 CPU 负载这么高，响应时间低，那么可以推测及有可能与 Java 线程有关系，于是通过 `jstack -l [java pid]`，发现有将近百多的 HTTP 线程其 State 都处于 RUNNABLE，这不是问题，问题在于基本上所有的线程都在执行同一方法及同一代码行： 

{% highlight java %}
"http-8068-52" daemon prio=10 tid=0x000000005cac5800 nid=0x280d runnable [0x0000000046060000] 
   java.lang.Thread.State: RUNNABLE 
        at java.net.SocketInputStream.socketRead0(Native Method) 
        at java.net.SocketInputStream.read(SocketInputStream.java:129) 
        at com.mapbar.logic.util.SocketUtil.getResponse(SocketUtil.java:63) 
        at com.mapbar.logic.geocoding.GeoCoding.getLonLat(GeoCoding.java:73) 
  "http-8068-51" daemon prio=10 tid=0x000000005ccbe000 nid=0x280c runnable [0x0000000045f5f000] 
   java.lang.Thread.State: RUNNABLE 
        at java.net.SocketInputStream.socketRead0(Native Method) 
        at java.net.SocketInputStream.read(SocketInputStream.java:129) 
        at com.mapbar.logic.util.SocketUtil.getResponse(SocketUtil.java:63) 
        at com.mapbar.logic.geocoding.GeoCoding.getLonLat(GeoCoding.java:73) 
{% endhighlight java %}

于是可以断定代码 SocketUtil.getResponse 中的 Socket 调用几乎是肯定的罪魁祸首。 


#### References
* [Java Thread Dumps ](http://java.sys-con.com/node/1611555)
* [An Introduction to Java Stack Traces](http://java.sun.com/developer/technicalArticles/Programming/Stacktrace/)
* [java.net.socketinputstream.socketread0 hangs thread ](http://javaeesupportpatterns.blogspot.com/2011/04/javanetsocketinputstreamsocketread0.html)


<hr/>

####CASE

因最近在使用 HBase，但是 HBase 与 Hadoop 集成一直存在问题，主要原因是 Hadoop 0.20.x 版本本身不支持 append 操作，以至于 HBase 中包括了 hadoop-core-0.20-append-r1056497.jar 的支持 append 操作的 Hadoop 版本。不过后来 Hadoop 发出了hadoop-0.20.205.0版本，而且已经拥有了 append 的支持。于是我便将该版本的 Hadoop 与 hbase-0.90.5 进行整合，启动时一直提示：
>Caused by: java.lang.ClassNotFoundException: org.apache.commons.configuration.Configuration

出现以上异常，导致 HMaster 无法正常启动。 

#### Solved 

1. 刪除 {hbase_home}/lib/hadoop-core-0.20-append-r1056497.jar 
2. 將 {hadoop_home}/hadoop-core-0.20.205.0.jar复制到 {hbase_home}/lib/ 
3. 將 {hadoop_home}/commons-configuration-1.6.jar复制到 {hbase_home}/lib/ 

如果你的 hadoop_home 没有 commons-configuration-1.6.jar的话可以点[这里](http://repo1.maven.org/maven2/commons-configuration/commons-configuration/1.6/commons-configuration-1.6.jar)下载。 
重新启动 HBase 即可解决以上异常。
