## Java

### 基础

### 集合

### 并发
[ForkJoinPool详解](docs/java/concurrent/ForkJoinPool源码解析/ForkJoinPool源码详解.md)

### JVM

### 新特性

## 计算机基础

### 操作系统

### 网络

- 电脑访问互联网某台服务器域名时，是怎么知道它的IP地址的？（DNS域名解析） 
- 电脑访问互联网某台服务器的IP地址时，是怎么知道它的MAC地址的？（ARP/代理ARP协议） [参考](https://www.bilibili.com/video/BV1jV4y1G77w/?spm_id_from=autoNext&vd_source=099fd42798be92bb9ffab7824eb5b945)
- 电脑访问互联网某台服务器时，通信数据包如何将其从局域网发送到互联网？（NAT地址翻转）[参考](http://www.52im.net/thread-3506-1-1.html?spm=a2c6h.12873639.article-detail.16.570f3ab0TTsNAj) 
- 电脑访问互联网某台服务器时，通信数据包怎么知道目标IP具体的位置？（路由协议，例如内网跑OSPF，外网跑ISIS和BGP） 
- 电脑访问互联网某台服务器时，会采用哪种应用协议跟终端通信，数据包怎么封装的？（这取决于电脑采用的软件，如果用Ping，则是ICMP，如果是QQ，则是OICQ，如果是微信，则是http/tcp）

[Wireshark介绍及抓包分析并分析tcp的连接过程](https://pdai.tech/md/develop/protocol/dev-protocol-tool-wireshark.html)

[NAT 是如何工作的](https://blog.51cto.com/u_15239532/3009528)

[TCP连接请求过程(含keepalive)](docs/cs-basics/network/TCP连接请求过程(含keepalive).md)

[TCP以及HTTP长连接](docs/cs-basics/network/TCP以及HTTP的长连接.md)

[拔掉网线再插上，TCP连接还在吗？](https://developer.aliyun.com/article/875118)


### 数据结构

### 算法

## 数据库

### MySQL

### redis

## 源码分析

### 数据库连接源码解析

[Druid数据库连接池源码解析](docs/source-code-analysis/数据库连接/Druid数据库连接池源码解析.md)

[Mysql的JDBC驱动源码解析](docs/source-code-analysis/数据库连接/Mysql的JDBC驱动源码解析.md)


### Tomcat源码解析

[Tomcat源码解析系列一：Tomcat的整体架构](docs/source-code-analysis/tomcat/系列一：Tomcat的整体架构.md)

[Tomcat源码解析系列二：Tomcat类加载机制以及Context的初始化](docs/source-code-analysis/tomcat/系列二：Tomcat类加载机制以及Context的初始化.md)

[Tomcat源码解析系列三：Tomcat的Reactor机制-http请求连接处理机制](docs/source-code-analysis/tomcat/系列三：Tomcat的Reactor机制-http请求连接处理机制.md)

[Tomcat源码解析系列四：Tomcat的url到Wrapper的映射](docs/source-code-analysis/tomcat/系列四：Tomcat的url到Wrapper的映射.md)

[Tomcat源码解析系列五：Tomcat处理Http请求](docs/source-code-analysis/tomcat/系列五：Tomcat处理Http请求.md)

[Tomcat处理请求的编解码](docs/source-code-analysis/tomcat/Tomcat处理请求的编解码.md)

### Netty源码解析

[Netty源码解析系列一：Netty架构](docs/source-code-analysis/Netty/系列一：Netty架构.md)

[Netty源码解析系列二：Netty请求的处理流程](docs/source-code-analysis/Netty/系列二：Netty请求的处理流程.md)

[Netty源码解析系列三：Netty与Tomcat的区别](docs/source-code-analysis/Netty/系列三：Netty与Tomcat的区别.md)

[Netty源码解析系列四：Netty的长连接以及心跳](docs/source-code-analysis/Netty/Netty的长连接以及心跳.md)

### SpringMVC源码解析

[SpringMVC源码解析系列一：SpringMVC请求处理流程](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列一：SpringMVC请求处理流程.md)

[SpringMVC源码解析系列二：SpringMVC的HandlerMapping](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列二：SpringMVC的HandlerMapping.md)

[SpringMVC源码解析系列三：SpringMVC的参数解析-HandlerMethodArgumentResolver处理流程](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列三：SpringMVC的参数解析-HandlerMethodArgumentResolver处理流程.md)

[SpringMVC源码解析系列四：SpringMVC的返回值解析-HandlerMethodReturnValueHandler处理流程](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列四：SpringMVC的返回值解析-HandlerMethodReturnValueHandler处理流程.md)

[SpringMVC源码解析系列五：SpringMVC的异常处理-HandlerExceptionResolver处理流程](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列五：SpringMVC的异常处理-HandlerExceptionResolver处理流程.md)

[SpringMVC源码解析系列六：RequestMappingHandlerMapping的初始化](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列六：RequestMappingHandlerMapping的初始化.md)

[SpringMVC源码解析系列七：RequestMappingHandlerAdapter的初始化](docs/source-code-analysis/Spring/SpringMVC/SpringMVC系列七：RequestMappingHandlerAdapter的初始化.md)

### Spring源码解析

[Spring源码解析系列一：Spring的IOC的Bean生命周期源码详解](docs/source-code-analysis/Spring/Spring的IOC的Bean生命周期源码详解.md)

[Spring源码解析系列二：Spring的IOC容器实例化Bean的方式源码详解](docs/source-code-analysis/Spring/Spring的IOC容器实例化Bean的方式源码详解.md)

[Spring源码解析系列三：Spring的IOC容器的属性注入源码详解](docs/source-code-analysis/Spring/Spring的IOC容器的属性注入源码详解.md)

[Spring源码解析系列四：Spring的AOP源码详解](docs/source-code-analysis/Spring/Spring的AOP源码详解.md)

[Spring源码解析系列五：Spring的AOP之动态代理源码详解](docs/source-code-analysis/Spring/Spring的AOP之动态代理源码详解.md)

[Spring源码解析系列六：Spring的事务结合Mybatis源码详解](docs/source-code-analysis/Spring/Spring的事务结合Mybatis源码详解.md)

[Spring的${}以及#{}的源码处理](docs/source-code-analysis/Spring/Spring的"$"{}以及#{}的源码处理.md)

[Spring的BeanPostProcessor的顺序](docs/source-code-analysis/Spring/Spring的BeanPostProcessor的顺序.md)

[Spring结合Tomcat解决Jsp乱码问题](docs/source-code-analysis/Spring/Spring结合Tomcat解决Jsp乱码问题.md)

## 开发工具

### Git

### Docker

