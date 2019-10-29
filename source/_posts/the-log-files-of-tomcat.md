---
title: tomcat下各日志文件的说明
path: the-log-files-of-tomcat
date: 2017-04-15 17:00:37
updated: 2017-04-15 17:00:37
categories:
- Tomcat
tags:
- java
- tomcat
- spring
---

&emsp;&emsp;今天同事在使用tomcat发布项目的时候，遇到个问题，导致项目一直无法启动，查看tomcat控制台输出，发现启动日志仅有一句描述：`严重: One or more listeners failed to start. Full details will be found in the appropriate container log file`，第一时间认为是spring容器初始化失败，那么把spring加载配置文件及初始化的日志级别调整为debug不就万事大吉了么。<!--more-->
&emsp;&emsp;然并卵，日志输出中还是仅有那一句描述，使用tomcat-manager卸载-发布-卸载-发布多个循环，仍无果。  
&emsp;&emsp;仔细思考想想，最近查问题都是使用catalina.out以及项目日志来排查，tomcat的其他日志中是不是藏有猫腻。  
&emsp;&emsp;最终在`localhost.*.log`中发现了问题，项目中spring容器扫描package时加载了不同包名但类名相同的bean，导致初始化监听器失败，可是为什么应用的日志组件已经启动，为什么没有报呢？还是翻翻spring、tomcat的源码瞧个仔细吧。  
经查阅，spring中contextloadlistener中的异常都是抛出到web容器的，由容器来进行处理，那么就可以定位到tomcat中[standardcontext中初始化listener的位置](http://svn.apache.org/repos/asf/tomcat/tc8.5.x/trunk/java/org/apache/catalina/core/StandardContext.java)，摘录如下：  
    
``` java
for (int i = 0; i < instances.length; i++) {
    if (!(instances[i] instanceof ServletContextListener))
        continue;
    ServletContextListener listener =
        (ServletContextListener) instances[i];
    try {
        fireContainerEvent("beforeContextInitialized", listener);
        if (noPluggabilityListeners.contains(listener)) {
            listener.contextInitialized(tldEvent);
        } else {
            listener.contextInitialized(event);
        }
        fireContainerEvent("afterContextInitialized", listener);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        fireContainerEvent("afterContextInitialized", listener);
        getLogger().error
            (sm.getString("standardContext.listenerStart", instances[i].getClass().getName()), t);
        ok = false;
    }
}
```
&emsp;&emsp;好吧，如此可知道日志是通过standardcontext的父类ContainerBase来输出的，顺便瞅一眼tomcat的日志配置，默认为`conf/logging.properties`，看到该类的日志被输出到location.*.log中，与文首场景中对应上了。
&emsp;&emsp;这么简单的问题查了小半天，惆怅。
&emsp;&emsp;tomcat的相关文档中提示：调用`javax.servlet.ServletContext.log(...)`输出日志会被tomcat内部日志组件处理，按照日志配置文件中`org.apache.catalina.core.ContainerBase.[${engine}].[${host}].[${context}]` 的形式被分类记录，那我们顺便了解下tomcat默认的日志配置吧。
    
    
>1catalina.org.apache.juli.FileHandler.level = FINE
>1catalina.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
>1catalina.org.apache.juli.FileHandler.prefix = catalina.
>
>2localhost.org.apache.juli.FileHandler.level = FINE
>2localhost.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
>2localhost.org.apache.juli.FileHandler.prefix = localhost.
>
>3manager.org.apache.juli.FileHandler.level = FINE
>3manager.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
>3manager.org.apache.juli.FileHandler.prefix = manager.
>
>4host-manager.org.apache.juli.FileHandler.level = FINE
>4host-manager.org.apache.juli.FileHandler.directory = ${catalina.base}/logs
>4host-manager.org.apache.juli.FileHandler.prefix = host-manager.
>    

&emsp;&emsp;另外，在`conf/server.xml`中我们可以开启请求记录日志，那么可以输出五个文件：
1.catalina.yyyy-MM-dd.log  Cataline引擎的日志文件，记录启动的JVM参数以及操作系统等信息  
2.localhost.yyyy-MM-dd.log  tomcat中名为localhost的host日志输出，记录host初始化的日志
3.localhost_access_log.yyyy-MM-dd.txt  记录localhost下各应用的请求
4.host-manager.yyyy-MM-dd.log  记录localhost下[host-manager](http://tomcat.apache.org/tomcat-8.5-doc/host-manager-howto.html)的应用日志
5.manager.yyyy-MM-dd.log  记录localhost下[manager](http://tomcat.apache.org/tomcat-8.5-doc/manager-howto.html)的应用日志