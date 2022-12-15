# SpringMVC的请求处理流程

[TOC]



## 核心流程

### DispatcherServlet 接收到客户端发送的请求

调用doService方法->调用doDispatch方法

### 组装HandlerExecutionChain

#### 通过RequestMappingHandlerMapping获得HandlerMethod

#### 添加匹配的HandlerInterceptor

### 获取HandlerAdapter：RequestMappingHandlerAdapter

### 调用HandlerInterceptor的preHandle方法

### 真正的参数解析、方法处理、返回值处理的逻辑

#### 调用@ModelAttribute标注的方法，并把信息存到request维度的ModelAndViewContainer的Model中

#### 使用HandlerMethodArgumentResolver的resolveArgument来解析参数

#### 反射调用方法

#### 使用HandlerMethodReturnValueHandler的handleReturnValue来处理返回值





### 调用HandlerInterceptor的postHandle方法

### 调用HandlerExceptionResolver的resolveException方法来处理异常（此时请求处理的所有异常）

### 调用DispatcherServlet的render来渲染视图

### 调用HandlerInterceptor的afterCompletion方法，异常是（resolveException导致、解析页面导致的异常）























## 总结

1. DispatcherServlet 接收到客户端发送的请求
2. HandleMapping 根据请求 URL 找到对应的 HandlerMethod 以及 HandlerInterceptor，得到 HandlerExecutionChain
3. 根据 HandlerMethod 获取 HandlerAdapter
4. 顺序调用 HandlerInterceptor 的 preHandle 方法
5. 调用@ModelAttribute标注的方法
6. 利用HandlerMethodArgumentResolver进行参数解析，其中需要使用spring进行类型转换的还会先调用 @InitBinder 标注的方法
7. 反射调用方法
8. 利用HandlerMethodReturnValueHandler进行返回值处理，其中@ResponseBody标注方法直接将数据刷到网络中，普通的是Tomcat的 Servlet 处理完之后由 Tomcat 把数据刷到网络上
9. 逆序调用 HandlerInterceptor 的 postHandle 方法
10. 调用 HandlerExceptionResolve r的 resolveException 方法来处理异常，其中要重复 6-8 步骤进行异常方法的调用
11. 渲染视图，把 model 中的数据放入到 request 的 attribute 中
12. DispatcherServlet 将得到的视图进行渲染，填充到request域中
13. 逆序调用 HandlerInterceptor 的 afterCompletion 方法













