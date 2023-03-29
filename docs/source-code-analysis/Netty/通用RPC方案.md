# 通用RPC方案

## provider端

1. 利用org.springframework.context.ApplicationListener#onApplicationEvent在Spring初始化的最后一步时绑定端口
2. 利用Netty将ServerSocketChannel绑定到boss 线程绑定端口

```java
((ServerBootstrap)this.serverBootstrap
.group(getParentEventLoopGroup(), getChildEventLoopGroup())
.channel(NioServerSocketChannel.class)
.childHandler(new ServerChannelInitializer(this.config));

((ServerBootstrap)((ServerBootstrap)((ServerBootstrap)((ServerBootstrap)((ServerBootstrap)this.serverBootstrap
.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, this.config.getCONNECTTIMEOUT()))
.option(ChannelOption.SO_BACKLOG, this.config.getBACKLOG()))
.option(ChannelOption.ALLOCATOR, PooledBufHolder.getInstance()))
.option(ChannelOption.SO_REUSEADDR, reusePort))
.option(ChannelOption.RCVBUF_ALLOCATOR, AdaptiveRecvByteBufAllocator.DEFAULT))
.childOption(ChannelOption.SO_KEEPALIVE, this.config.isKEEPALIVE())
.childOption(ChannelOption.TCP_NODELAY, this.config.isTCPNODELAY())
.childOption(ChannelOption.ALLOCATOR, PooledBufHolder.getInstance())
.childOption(ChannelOption.SO_RCVBUF, 1048576)
.childOption(ChannelOption.SO_SNDBUF, 1048576)
.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, this.config.getBufferLowWaterMark())
.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, this.config.getBufferHighWaterMark());

ChannelFuture future = this.serverBootstrap.bind(new InetSocketAddress(this.config.getHost(), this.config.getPort()));
```

其中ServerChannelInitializer的initChannel为

```java
protected void initChannel(SocketChannel ch) throws Exception {
    ch.pipeline().addLast(new ChannelHandler[]{this.connectionChannelHandler}).addLast(new ChannelHandler[]{new SerializeAdapterDecoder(this.serverChannelHandler, this.transportConfig.getPayload(), this.transportConfig.isTelnet(), this.transportConfig)});
}
```

触发read事件时，在SerializeAdapterDecoder的decode方法的最后一个中加入ServerChannelHandler 继承 ChannelInboundHandlerAdapter

继续处理read事件时，在ServerChannelHandler 的channelRead方法中使用线程池来处理读到的数据（可能是耗时操作）

从而来保证不会由于处理请求导致work线程全部耗尽，而出现等待的情况

### 小结

1. 利用org.springframework.context.ApplicationListener#onApplicationEvent在Spring初始化的最后一步时绑定端口
2. 利用Netty将ServerSocketChannel绑定到work 线程绑定端口
3. consumer端连接到provider端
4. 将接收的SocketChannel绑定到work 线程中
5. consumer端传输数据给provider端
6. provider端处理读事件，在pipeline的最后一个ChannelInboundHandlerAdapter时，使用线程池来处理读到的数据（可能是耗时操作）

## consumer端

1. 使用FactoryBean在Spring初始化Bean的时候代理得到真正的代理Bean

2. 从注册中心获取所有的provider

3. 在线程池中利用Netty连接到provider

   

   ```java
   EventLoopGroup eventLoopGroup = transportConfig.getEventLoopGroup();
   Channel channel = null;
   String host = transportConfig.getProvider().getIp();
   int port = transportConfig.getProvider().getPort();
   int connectTimeout = transportConfig.getConnectionTimeout();
   
   Bootstrap bootstrap = new Bootstrap();
   ((Bootstrap)bootstrap.group(eventLoopGroup)).channel(NioSocketChannel.class);
   
   bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
   bootstrap.option(ChannelOption.ALLOCATOR, PooledBufHolder.getInstance());
   bootstrap.option(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, transportConfig.getLowWaterMark());
   bootstrap.option(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, transportConfig.getHighWaterMark());
   bootstrap.option(ChannelOption.RCVBUF_ALLOCATOR, AdaptiveRecvByteBufAllocator.DEFAULT);
   ClientChannelInitializer initializer = new ClientChannelInitializer(transportConfig);
   bootstrap.handler(initializer);
   ChannelFuture channelFuture = bootstrap.connect(host, port);
   channelFuture.awaitUninterruptibly((long)connectTimeout, TimeUnit.MILLISECONDS);
   if (channelFuture.isSuccess()) {
       channel = channelFuture.channel();
       if (NetUtils.toAddressString((InetSocketAddress)channel.remoteAddress())
           .equals(NetUtils.toAddressString((InetSocketAddress)channel.localAddress()))) {
           channel.close();
           throw new InitErrorException("Failed to connect " + host + ":" + port + ". Cause by: Remote and local address are the same");
       } else {
           return channel;
       }
   } else {
       Throwable cause = channelFuture.cause();
       throw new InitErrorException("Failed to connect " + host + ":" + port + (cause != null ? ". Cause by: " + cause.getMessage() : "."));
   }
   ```

其中ClientChannelInitializer中加入编码器以及解码器进行编解码通信，以及ClientChannelHandler来处理请求的msgId与响应的msgId的对应关系



真正调用是使用负载均衡器获取到provider

得到provider对应的NioSocketChannel

真正发消息（含有readTimeout读超时时间）：

```java
msgId = this.genarateRequestId(msg);
msg.setRequestId(msgId);
MsgFuture<ResponseMessage> future = this.doSendAsyn(msg, timeout);
var5 = (ResponseMessage)future.get((long)timeout, TimeUnit.MILLISECONDS);
```

在ClientChannelHandler的channelRead处理read事件：

```java
public void receiveResponse(ResponseMessage msg) {
    Integer msgId = msg.getRequestId();
    MsgFuture future = (MsgFuture)this.futureMap.remove(msgId);
    if (future == null) {
        if (msg != null && msg.getMsgBody() != null) {
            msg.getMsgBody().release();
        }
    } else {
        future.setSuccess(msg);
    }
}
```

