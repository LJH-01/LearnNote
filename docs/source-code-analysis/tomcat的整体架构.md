# tomcat的整体架构

## 架构图



Tomcat中最顶层的容器是Server，代表着整个服务器，从上图中可以看出，一个Server可以包含至少一个Service，用于具体提供服务。

Service主要包含两个部分：Connector和Container。从上图中可以看出 Tomcat 的心脏就是这两个组件，他们的作用如下：

1、Connector用于处理连接相关的事情，并提供Socket与Request和Response相关的转化;
2、Container用于封装和管理Servlet，以及具体处理Request请求；

一个Tomcat中只有一个Server，一个Server可以包含多个Service，一个Service只有一个Container，但是可以有多个Connectors，这是因为一个服务可以有多个连接，如同时提供Http和Https链接，也可以提供向相同协议不同端口的连接,示意图如下（Engine、Host、Context下边会说到）：


![image-20220718151157989](assets/image-20220718151157989.png)


![image-20220718151214843](assets/image-20220718151214843.png)

## 架构小结





## tomcat启动流程

![image-20220719102639459](assets/image-20220719102639459.png)



**启动步骤**

启动tomcat ， 需要调用 bin/startup.bat (在linux 目录下 , 需要调用 bin/startup.sh) ， 在startup.bat 脚本中, 调用了catalina.bat。
在catalina.bat 脚本文件中，调用了BootStrap 中的main方法。
在BootStrap 的main 方法中调用了 init 方法 ， 来创建Catalina 及 初始化类加载器。
在BootStrap 的main 方法中调用了 load 方法 ， 在其中又调用了Catalina的load方 法。
在Catalina 的load 方法中 , 需要进行一些初始化的工作, 并需要构造Digester 对象, 用 于解析 XML。
然后在调用后续组件的初始化操作 。。。 加载Tomcat的配置文件，初始化容器组件 ，监听对应的端口号， 准备接受客户端请求。

在context的start方法中初始化WebappLoader，在WebappLoader初始化时赋值   `private String loaderClass = ParallelWebappClassLoader.class.getName();` 并调用WebappLoader的start()方法，在start()方法中实例化类加载器为ParallelWebappClassLoader

## tomcat的Lifecycle接口

Tomcat所有的组件均存在初始化、启动、停止等生命周期方法，拥有生命周期管理的特性， 所以Tomcat在设计的时候， 基于生命周期管理抽象成了一个接口 Lifecycle 。而组件 Server、Service、Container、Executor、Connector 组件 ， 都实现了一个生命周期的接口。

**生命周期中的核心方法**

- init（）：初始化组件
- start（）：启动组件
- stop（）：停止组件
- destroy（）：销毁组件





## tomcat重要组件的默认实现

| 接口                                                       | 默认实现                                                     | 阀值                 |
| ---------------------------------------------------------- | ------------------------------------------------------------ | -------------------- |
| Server                                                     | StanderdServer                                               |                      |
| Service                                                    | StanderdService                                              |                      |
| Engine                                                     | StanderdEngine                                               | StandardEngineValve  |
| Host                                                       | StanderdHost                                                 | StandardHostValve    |
| Context                                                    | StanderdContext                                              | StandardContextValve |
|                                                            | StandardWrapper                                              | StandardWrapperValve |
| Endpoint组件没有接口<br />但提供一个抽象类AbstractEndpoint | `<Connector protocol="HTTP/1.1"/>`<br />使用NioEndpoint      |                      |
| ProtocolHandler                                            | `<Connector protocol="HTTP/1.1"/>`<br />使用Http11NioProtocol |                      |
| Processor                                                  | `<Connector protocol="HTTP/1.1"/>`<br />使用Http11Processor  |                      |



基础阀org.apache.catalina.core.StandardWrapperValve的invoke方法，在这里最终会调用请求的url所匹配的Servlet相关过滤器（filter）的doFilter方法及该Servlet的service方法（这段实现都是在过滤器链ApplicationFilterChain类的doFilter方法中）

 这里可以看出容器内的Engine、Host、Context、Wrapper容器组件的实现的共通点：

1.这些组件内部都有一个成员变量pipeline，因为它们都是从org.apache.catalina.core.ContainerBase类继承来的，pipeline就定义在这个类中。所以每一个容器内部都关联了一个管道。

2.都是在类的构造方法中设置管道内的基础阀。

3.所有的基础阀的实现最后都会调用其下一级容器（直接从请求中获取下一级容器对象的引用，在前一篇文章的分析中已经设置了与该请求相关的各级具体组件的引用）的getPipeline().getFirst().invoke()方法，直到Wrapper组件。因为Wrapper是对一个Servlet的包装，所以它的基础阀内部调用的过滤器链的doFilter方法和Servlet的service方法。

Tomcat8.5版本中，默认采用的是 NioEndpoint。

![image-20220719105126250](assets/image-20220719105126250.png)

## 参考文章
- https://blog.csdn.net/xlgen157387/article/details/79006434
- https://blog.csdn.net/lingyiwin/article/details/125428376

