

# Netty请求的处理流程



[TOC]





## 简单使用

### 服务端绑定端口并处理请求

```java
public class NettyServer {

    public static void main(String[] args) {
        new NettyServer().bing(7397);
    }

    private void bing(int port) {
        //配置服务端NIO线程组
        EventLoopGroup parentGroup = new NioEventLoopGroup(1); //NioEventLoopGroup extends MultithreadEventLoopGroup Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
        EventLoopGroup childGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(parentGroup, childGroup)
                    .channel(NioServerSocketChannel.class)    //非阻塞模式
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel channel) throws Exception {
                            //对象传输处理
                            channel.pipeline().addLast(new ObjDecoder(MsgInfo.class));
                            channel.pipeline().addLast(new ObjEncoder(MsgInfo.class));
                            // 在管道中添加我们自己的接收数据实现方法
                            channel.pipeline().addLast(new LoggingHandler());
                            channel.pipeline().addLast(new MyServerHandler());
                        }
                    });
            ChannelFuture f = b.bind(port).sync();
            System.out.println(" demo-netty server start done. ");
            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            childGroup.shutdownGracefully();
            parentGroup.shutdownGracefully();
        }

    }

}
```

### 客户端连接服务端

```java
public class NettyClient {

    public static void main(String[] args) {
        new NettyClient().connect("127.0.0.1", 7397);
    }

    private void connect(String inetHost, int inetPort) {
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(workerGroup);
            b.channel(NioSocketChannel.class);
            b.option(ChannelOption.AUTO_READ, true);
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel channel) throws Exception {
                    //对象传输处理
                    channel.pipeline().addLast(new ObjDecoder(MsgInfo.class));
                    channel.pipeline().addLast(new ObjEncoder(MsgInfo.class));
                    // 在管道中添加我们自己的接收数据实现方法
                    channel.pipeline().addLast(new MyClientHandler());
                }
            });
            ChannelFuture f = b.connect(inetHost, inetPort).sync();
            System.out.println("demo-netty client start done.");

            f.channel().writeAndFlush(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息1"));
            f.channel().write(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息2"));

            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }

}
```

## NioEventLoop进行的操作

### 处理连接以及处理事件

```java
protected void run() {
    // Netty解决办法具体步骤：
    //
    // 1、先定义当前时间currentTimeNanos。
    // 2、接着计算出一个执行最少需要的时间timeoutMillis。
    // 3、每次对selectCnt做++操作。
    // 4、进行判断，如果到达执行到最少时间，则seletCnt重置为1。
    // 5、一旦到达SELECTOR_AUTO_REBUILD_THRESHOLD这个阀值，就需要重建selector来解决这个问题。
    // 6、这个阀值默认是512。
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                // selectStrategy 终于要派上用场了
                // 它有两个值，一个是 CONTINUE 一个是 SELECT
                // 针对这块代码，我们分析一下。
                // 1. 如果 taskQueue 不为空，也就是 hasTasks() 返回 true，
                //         那么执行一次 selectNow()，该方法不会阻塞
                // 2. 如果 hasTasks() 返回 false，那么执行 SelectStrategy.SELECT 分支，
                //    进行 select(...)，这块是带阻塞的
                // 这个很好理解，就是按照是否有任务在排队来决定是否可以进行阻塞
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; // nothing on the calendar
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary selector wakeups
                        // so use of lazySet is ok (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            // 默认地，ioRatio 的值是 50
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                // 如果 ioRatio 不是 100，那么根据 IO 操作耗时，限制非 IO 操作耗时
                final long ioStartTime = System.nanoTime();
                try {
                    // 执行 IO 操作
                    processSelectedKeys();
                } finally {
                    // 根据 IO 操作消耗的时间，计算执行非 IO 操作（runAllTasks）可以用多少时间.
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }

            // 渠道任务设置 selectCnt 为 0
            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
                selectCnt = 0;
                // 避免jdk空轮询的bug
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
        } catch (Error e) {
            throw e;
        } catch (Throwable t) {
            handleLoopException(t);
        } finally {
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Error e) {
                throw e;
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}
```

### 解决空轮训cpu100%的bug



```java
private boolean unexpectedSelectorWakeup(int selectCnt) {
    if (Thread.interrupted()) {
        // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
        // As this is most likely a bug in the handler of the user or it's client library we will
        // also log it.
        //
        // See https://github.com/netty/netty/issues/2426
        if (logger.isDebugEnabled()) {
            logger.debug("Selector.select() returned prematurely because " +
                    "Thread.currentThread().interrupt() was called. Use " +
                    "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
        }
        return true;
    }
    // select()没有阻塞立即返回了, 可能触发了空轮询, 这个值是512, 也就是说512次轮询的结果都为空
    if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
        // The selector returned prematurely many times in a row.
        // Rebuild the selector to work around the problem.
        logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                selectCnt, selector);
        // 把老的selectedKeys都注册到一个新的selector里面去, 并替换当前的selector
        rebuildSelector();
        return true;
    }
    return false;
}
```



```java
public void rebuildSelector() {
    if (!inEventLoop()) {
        execute(new Runnable() {
            @Override
            public void run() {
                // 把老的selectedKeys都注册到一个新的selector里面去, 替换当前的selector
                rebuildSelector0();
            }
        });
        return;
    }
    // 把老的selectedKeys都注册到一个新的selector里面去, 替换当前的selector
    rebuildSelector0();
}
```

```java
private void rebuildSelector0() {
    final Selector oldSelector = selector;
    final SelectorTuple newSelectorTuple;

    if (oldSelector == null) {
        return;
    }

    try {
        newSelectorTuple = openSelector();
    } catch (Exception e) {
        logger.warn("Failed to create a new Selector.", e);
        return;
    }

    // Register all channels to the new Selector.
    int nChannels = 0;
    // 转移 SelectionKey
    for (SelectionKey key: oldSelector.keys()) {
        Object a = key.attachment();
        try {
            if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                continue;
            }

            int interestOps = key.interestOps();
            key.cancel();
            SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
            if (a instanceof AbstractNioChannel) {
                // Update SelectionKey
                ((AbstractNioChannel) a).selectionKey = newKey;
            }
            nChannels ++;
        } catch (Exception e) {
            logger.warn("Failed to re-register a Channel to the new Selector.", e);
            if (a instanceof AbstractNioChannel) {
                AbstractNioChannel ch = (AbstractNioChannel) a;
                ch.unsafe().close(ch.unsafe().voidPromise());
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                invokeChannelUnregistered(task, key, e);
            }
        }
    }

    // 设置 selector
    selector = newSelectorTuple.selector;
    unwrappedSelector = newSelectorTuple.unwrappedSelector;

    try {
        // time to close the old selector as everything else is registered to the new one
        oldSelector.close();
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("Failed to close the old Selector.", t);
        }
    }

    if (logger.isInfoEnabled()) {
        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }

```



## 处理流程

### 1.server端绑定端口

```java
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}
```

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 初始化 NioServerSocketChannel 或者 NioSocketChannel
    // 组装好 pipeline 中要添加的 ChannelHandler, 并回调对应的事件
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();
                    // 回调绑定端口
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 初始化 NioServerSocketChannel 或者 NioSocketChannel
        channel = channelFactory.newChannel();
        // 在 pipeline 中添加 ChannelInitializer
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    // 组装好 pipeline 中要添加的 ChannelHandler, 并回调对应的事件
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}
```

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    // 组装好 pipeline 中要添加的 ChannelHandler, 并回调对应的事件
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        // 进行 JDK 底层的操作：Channel 注册到 Selector 上
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        // 调用 ChannelInitializer 的 init(channel)
        // 我们之前说过，init 方法会将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中
        pipeline.invokeHandlerAddedIfNeeded();

        // 设置当前 promise 的状态为 success
        //   因为当前 register 方法是在 eventLoop 中的线程中执行的，需要通知提交 register 操作的线程
        safeSetSuccess(promise);
        // 当前的 register 操作已经成功，该事件应该被 pipeline 上
        //   所有关心 register 事件的 handler 感知到，往 pipeline 中扔一个事件
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        // 这里 active 指的是 channel 已经打开，保证 fireChannelActive 只执行一次
        // true：ServerSocketChannel 绑定成功或者 SocketChannel 连接成功
        if (isActive()) {
            // 如果该 channel 是第一次执行 register，那么 fire ChannelActive 事件
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                // 该 channel 之前已经 register 过了，
                // 这里让该 channel 立马去监听通道中的 OP_READ 事件
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

之后调 HeadContext 的 bind 调unsafe 的bind 进行端口绑定

```java
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
}
```

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        // 绑定端口
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    // 绑定端口之后, 调用 fireChannelActive
    // 在fireChannelActive 里面绑定监听的事件
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```

至此绑定端口成功

总结：

1. NioEventLoop.taskQueue中数据的变化
   1.  先加入AbstractChannel.AbstractUnsafe#register0，来组装好 pipeline 中要添加的 ChannelHandler, 并回调对应的事件
   2.  再加入ChannelOutboundInvoker#bind，来绑定端口
2. register0方法进行了如下操作
   1. Channel 注册到 Selector 上
   2. 将 ChannelInitializer 内部添加的 handler ：ServerBootstrapAcceptor 添加到 pipeline 中，最终该channel的pipeline中的 ChannelHandler已添加完成
   3. 回调 bind 端口，把绑定操作放到NioEventLoop.taskQueue中


### 2.server端在Boss NioEventLoop上注册accept事件

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        // 绑定端口
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    // 绑定端口之后, 调用 fireChannelActive
    // 在fireChannelActive 里面绑定监听的事件
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}

```

调用 DefaultChannelPipeline.HeadContext#channelActive 进行注册 accept 事件

```java
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();

    // 在 selector 上注册事件
    // NioServerSocketChannel 是 accept 事件
    // NioSocketChannel 是 read 事件
    readIfIsAutoRead();
}
```



最终调 unsafe 的 beginRead 绑定 accept 事件

```java
// 在 selector 上注册事件
// NioServerSocketChannel 是 accept 事件
// NioSocketChannel 是 read 事件
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

### 3.client端连接server端失败则注册connect事件

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 初始化 NioSocketChannel
    // 组装好 pipeline 中要添加的 ChannelHandler, 并回调对应的事件
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) {
        if (!regFuture.isSuccess()) {
            return regFuture;
        }
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                // Directly obtain the cause and do a null check so we only need one volatile read in case of a
                // failure.
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();
                    // 处理连接操作
                    doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

1. Channel 注册到 Selector 上
2. 将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中，最终该channel的pipeline中的 ChannelHandler已添加完成
3. 回调 doResolveAndConnect0，处理连接操作

```java
private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                           final SocketAddress localAddress, final ChannelPromise promise) {
    try {
        final EventLoop eventLoop = channel.eventLoop();
        AddressResolver<SocketAddress> resolver;
        try {
            resolver = this.resolver.getResolver(eventLoop);
        } catch (Throwable cause) {
            channel.close();
            return promise.setFailure(cause);
        }

        if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
            // Resolver has no idea about what to do with the specified remote address or it's resolved already.
            doConnect(remoteAddress, localAddress, promise);
            return promise;
        }

        final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);

        if (resolveFuture.isDone()) {
            final Throwable resolveFailureCause = resolveFuture.cause();

            if (resolveFailureCause != null) {
                // Failed to resolve immediately
                channel.close();
                promise.setFailure(resolveFailureCause);
            } else {
                // Succeeded to resolve immediately; cached? (or did a blocking lookup)
                // 进行连接操作
                doConnect(resolveFuture.getNow(), localAddress, promise);
            }
            return promise;
        }

        // Wait until the name resolution is finished.
        resolveFuture.addListener(new FutureListener<SocketAddress>() {
            @Override
            public void operationComplete(Future<SocketAddress> future) throws Exception {
                if (future.cause() != null) {
                    channel.close();
                    promise.setFailure(future.cause());
                } else {
                    doConnect(future.getNow(), localAddress, promise);
                }
            }
        });
    } catch (Throwable cause) {
        promise.tryFailure(cause);
    }
    return promise;
}
```

```java
private static void doConnect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (localAddress == null) {
                channel.connect(remoteAddress, connectPromise);
            } else {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```

最终调 DefaultChannelPipeline.HeadContext#connect 进行连接操作

```java
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```

调 unsafe.connect 连接服务端

```java
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null) {
            // Already a connect in process.
            throw new ConnectionPendingException();
        }

        boolean wasActive = isActive();
        // 进行连接操作
        if (doConnect(remoteAddress, localAddress)) {
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // Schedule connect timeout.
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        if (connectPromise != null && !connectPromise.isDone()
                                && connectPromise.tryFailure(new ConnectTimeoutException(
                                        "connection timed out: " + remoteAddress))) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

```java
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        doBind0(localAddress);
    }

    boolean success = false;
    try {
        // 连接操作
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {
            // 连接失败则注册 connect 事件
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```



进行连接操作，连接失败后，注册 connect 事件

总结：

1. Channel 注册到 Selector 上
2. 将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中，最终该channel的pipeline中的 ChannelHandler已添加完成
3. 回调 doResolveAndConnect0，处理连接操作
4. 进行连接操作，连接失败后，注册 connect 事件

### 4.server端收到accept事件，并在Work NioEventLoop上注册read事件

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            // 注册 read 事件
            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 处理 accept 以及 read 事件
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

调用io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read方法

```java
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    // 重置配置参数
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                // 调 ServerSocketChannel.accept 获取 SocketChannel
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (continueReading(allocHandle));
        } catch (Throwable t) {
            exception = t;
        }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            // 传播读事件，传递的是客户端通道
            // 在 ServerBootstrapAcceptor.channelRead 来注册获取到的SocketChannel的 read 事件到 work 线程上。
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            closed = closeOnReadError(exception);

            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            }
        }
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```

获取SocketChannel

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```

传播读事件，目前pipeline里面的channelhandler是ServerBootstrapAcceptor 调用 ServerBootstrapAcceptor.channelRead

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        // 将 SocketChannel 注册到 work NioEventLoop 的 Selector 上
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

```java
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();

    // 在 selector 上注册事件
    // NioServerSocketChannel 是 accept 事件
    // NioSocketChannel 是 read 事件
    readIfIsAutoRead();
}
```

```java
// 在 selector 上注册事件
// NioServerSocketChannel 是 accept 事件
// NioSocketChannel 是 read 事件
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```





总结：

1. 调用io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read方法
2. 调 ServerSocketChannel.accept 获取 SocketChannel
3. 传播读事件，目前pipeline里面的channelhandler是ServerBootstrapAcceptor 调用 ServerBootstrapAcceptor.channelRead
4. 将 SocketChannel 注册到 work NioEventLoop 的 Selector 上
5. 最终调 register0 调 pipeline.fireChannelActive()
6. 在 work NioEventLoop 的 Selector 上注册读事件

### 5.client端处理connect事件，并注册read事件

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            // 注册 read 事件
            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

```java
public final void finishConnect() {
    // Note this method is invoked by the event loop only if the connection attempt was
    // neither cancelled nor timed out.

    assert eventLoop().inEventLoop();

    try {
        boolean wasActive = isActive();
        // 校验 connect 成功
        doFinishConnect();
        // 注册 read 事件
        fulfillConnectPromise(connectPromise, wasActive);
    } catch (Throwable t) {
        fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
    } finally {
        // Check for null as the connectTimeoutFuture is only created if a connectTimeoutMillis > 0 is used
        // See https://github.com/netty/netty/issues/1770
        if (connectTimeoutFuture != null) {
            connectTimeoutFuture.cancel(false);
        }
        connectPromise = null;
    }
}
```

```java
private void fulfillConnectPromise(ChannelPromise promise, boolean wasActive) {
    if (promise == null) {
        // Closed via cancellation and the promise has been notified already.
        return;
    }

    // Get the state as trySuccess() may trigger an ChannelFutureListener that will close the Channel.
    // We still need to ensure we call fireChannelActive() in this case.
    boolean active = isActive();

    // trySuccess() will return false if a user cancelled the connection attempt.
    boolean promiseSet = promise.trySuccess();

    // Regardless if the connection attempt was cancelled, channelActive() event should be triggered,
    // because what happened is what happened.
    if (!wasActive && active) {
        // 注册 read 事件
        pipeline().fireChannelActive();
    }

    // If a user cancelled the connection attempt, close the channel, which is followed by channelInactive().
    if (!promiseSet) {
        close(voidPromise());
    }
}
```

```java
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();

    // 在 selector 上注册事件
    // NioServerSocketChannel 是 accept 事件
    // NioSocketChannel 是 read 事件
    readIfIsAutoRead();
}
```

```java
// 在 selector 上注册事件
// NioServerSocketChannel 是 accept 事件
// NioSocketChannel 是 read 事件
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
```

总结：

1. 处理 connnect 事件
2. 校验 connect 成功
3. 注册 read 事件

### 小结

**服务端处理流程：**

1. io.netty.channel.AbstractChannel.AbstractUnsafe#register0方法进行了如下操作
   1. Channel 注册到 Selector 上
   2. 将 ChannelInitializer 内部添加的 handler ：ServerBootstrapAcceptor 添加到 pipeline 中，最终该channel的pipeline中的 ChannelHandler已添加完成
   3. 回调 bind 端口，把绑定操作放到NioEventLoop.taskQueue中
2. io.netty.bootstrap.AbstractBootstrap#doBind0方法bind端口，并在select上注册 accept 事件
3. 在Boss NioEventLoop中处理accept事件，并将 SocketChannel 注册到 work NioEventLoop 的 Selector 上
   1. 调用io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read方法
   2. 调 ServerSocketChannel.accept 获取 SocketChannel
   3. 传播读事件，目前pipeline里面的channelhandler是ServerBootstrapAcceptor 调用 ServerBootstrapAcceptor.channelRead
   4. 将 SocketChannel 注册到 work NioEventLoop 的 Selector 上
   5. 最终调 register0 调 pipeline.fireChannelActive()
   6. 在 work NioEventLoop 的 Selector 上注册读事件

**客户端处理流程：**

1. Channel 注册到 Selector 上
2. 将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中，最终该channel的pipeline中的 ChannelHandler已添加完成
3. 回调 doResolveAndConnect0，处理连接操作
4. 进行连接操作，连接失败后，注册 connect 事件
5. 处理 connnect 事件
6. 校验 connect 成功
7. 注册 read 事件

**fireChannelActive的调用地方：**

服务端绑定selector成功后，在回调boss线程中调用的 fireChannelActive进行注册accept事件

服务端在收到accept事件之后，在work线程中调 register0 并在 register0 中调fireChannelActive进行注册read事件

客户端在接收到connect事件之后，调fireChannelActive进行注册read事件

### 处理read事件

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            // 注册 read 事件
            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 处理 accept 以及 read 事件
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

调用io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read方法

```java
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    // 每个channel对应一个 pipeline
    final ChannelPipeline pipeline = pipeline();
    // ByteBuf分配器
    final ByteBufAllocator allocator = config.getAllocator();
    // 容量计算器
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    // 重置，把之前计数的值全部清空
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            // 分配内存，关键在于计算分配内存的大小(小了不够，大了浪费)
            byteBuf = allocHandle.allocate(allocator);
            // doReadBytes,从socket读取字节到byteBuf,返回真实读取数量
            // 更新容量计算器
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                // 如果小于0则意味着socket关闭
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // 需要移除该渠道的 read 事件
                    readPending = false;
                }
                break;
            }

            // 增加循环计数器
            allocHandle.incMessagesRead(1);
            readPending = false;
            // 把读取到的数据，交给管道去处理
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
            // 判断是否继续从socket读取数据
        } while (allocHandle.continueReading());

        // 读取完成后调用readComplete，重新估算内存分配容量
        allocHandle.readComplete();
        // 继续注册 read 事件
        pipeline.fireChannelReadComplete();

        // 如果需要关闭，则处理关闭
        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        // 异常处理逻辑
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        // 根据情况移除OP_READ事件
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```

调用io.netty.handler.codec.ByteToMessageDecoder#channelRead方法

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 如果不是ByteBuf则不处理
    if (msg instanceof ByteBuf) {
        selfFiredChannelRead = true;
        // out用于存储解析二进制流得到的结果，一个二进制流可能会解析出多个消息，所以out是一个list
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            first = cumulation == null;
            cumulation = cumulator.cumulate(ctx.alloc(),
                    first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
            // 得到追加后的 cumulation 后，调用 decode 方法进行解码
            // 解码过程中，调用 fireChannelRead 方法，主要目的是将累积区的内容 decode 到 数组中
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            try {
                // 如果 cumulation 没有数据可读了，说明所有的二进制数据都被解析过了，此时对 cumulation 进行释放，
                // 以节省内存空间。
                // 反之 cumulation 还有数据可读，那么if中的语句不会运行，因为不对 cumulation 进行释放
                // 因此也就缓存了用户尚未解析的二进制数据。
                if (cumulation != null && !cumulation.isReadable()) {
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                    // 如果超过了 16 次，就压缩累计区，主要是将已经读过的数据丢弃，将 readIndex 归零。
                } else if (++numReads >= discardAfterReads) {
                    // We did enough reads already try to discard some bytes, so we not risk to see a OOME.
                    // See https://github.com/netty/netty/issues/4275
                    numReads = 0;
                    discardSomeReadBytes();
                }

                int size = out.size();
                // 如果没有向数组插入过任何数据
                firedChannelRead |= out.insertSinceRecycled();
                // 循环数组，向后面的 handler 发送数据，如果数组是空，那不会调用
                fireChannelRead(ctx, out, size);
            } finally {
                out.recycle();
            }
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        // 如果累计区还有可读字节
        while (in.isReadable()) {
            // 获取上一次decode方法调用后，out中元素数量，如果是第一次调用，则为0。
            final int outSize = out.size();

            // 上次循环成功, 解码
            if (outSize > 0) {
                // 用后面的业务 handler 的 ChannelRead 方法读取解析的数据
                fireChannelRead(ctx, out, outSize);
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                if (ctx.isRemoved()) {
                    break;
                }
            }

            int oldInputLength = in.readableBytes();
            // 在这里面回调 decode 方法，由开发者覆写，用于解析 in 中包含的二进制数据，并将解析结果放到out中。
            decodeRemovalReentryProtection(ctx, in, out);

            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) {
                break;
            }

            // 则说明当前decode方法调用没有解析出有效信息
            if (out.isEmpty()) {
                // 此时，如果发现上次decode方法和本次decode方法调用后，in中的剩余可读字节数相同
                // 则说明本次decode方法没有读取任何数据解析
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }

            // 处理人为失误 。如果走到这段代码，则说明outSize != out.size()。
            // 也就是本次decode方法实际上是解析出来了有效信息放到out中。
            // 但是oldInputLength == in.readableBytes()，说明本次decode方法调用并没有读取任何数据
            // 但是out中元素却添加了。
            // 这可能是因为开发者错误的编写了代码。
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```

```java
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception {
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
        // 回调 decode 方法，由开发者覆写，用于解析 in 中包含的二进制数据，并将解析结果放到out中
        decode(ctx, in, out);
    } finally {
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        decodeState = STATE_INIT;
        if (removePending) {
            fireChannelRead(ctx, out, out.size());
            out.clear();
            handlerRemoved(ctx);
        }
    }
}
```

调用用户自定义的decode方法，这里要考虑粘包拆包

```java
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    if (in.readableBytes() < 4) {
        // 拆包 继续读
        return;
    }
    in.markReaderIndex();
    int dataLength = in.readInt();
    if (in.readableBytes() < dataLength) {
        // 拆包 继续读
        in.resetReaderIndex();
        return;
    }
    byte[] data = new byte[dataLength];
    // 粘包 剩余数据不处理
    in.readBytes(data);
    out.add(SerializationUtil.deserialize(data, genericClass));
}
```

### 处理write操作

小例子：

```java
f.channel().writeAndFlush(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息1"))
        .addListener(new ResponseListener());
f.channel().write(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息2"))
        .addListener(new ResponseListener());
f.channel().write(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息3"))
        .addListener(new ResponseListener());
f.channel().write(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息4"))
        .addListener(new ResponseListener());
f.channel().writeAndFlush(MsgUtil.buildMsg(f.channel().id().toString(), "你好，这里是客户端发来的消息5"))
        .addListener(new ResponseListener());
```



```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ObjectUtil.checkNotNull(msg, "msg");
    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }

    final AbstractChannelHandlerContext next = findContextOutbound(flush ?
            (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            // 写数据并且回调 ChannelFutureListener
            next.invokeWriteAndFlush(m, promise);
        } else {
            // 写数据到缓存
            next.invokeWrite(m, promise);
        }
    } else {
        final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
        // 如果flush=true 写数据并且回调 ChannelFutureListener
        if (!safeExecute(executor, task, promise, m, !flush)) {
            // We failed to submit the WriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}
```

最终都会放到NioEventLoop的线程队列中等待执行

invokeWrite与 invokeWriteAndFlush的区别是：

- invokeWriteAndFlush是放入到jvm缓存中之后，马上把数据通过socket写出去
- invokeWrite是把数据放入到jvm缓存中之后，如果不调flush方法则数据不通过socket写出去

invokeWrite方法：

```java
void invokeWrite(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        // 写数据到缓存
        invokeWrite0(msg, promise);
    } else {
        write(msg, promise);
    }
}
```

invokeWriteAndFlush方法：

```java
void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        // 写数据到缓存
        invokeWrite0(msg, promise);
        // 真正写数据，并回调 ChannelFutureListener
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
```



先分析 invokeWrite0 方法：

调到 headcontext 的 write 方法

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}
```

```java
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();
    // 在数据被写出到网络上之前，所有待写出的数据都会存放在outboundBuffer中，这个类相当于发送缓存
    // 将来flush就是从outboundBuffer取出待写出数据写出到网络上
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        try {
            // release message now to prevent resource-leak
            ReferenceCountUtil.release(msg);
        } finally {
            // If the outboundBuffer is null we know the channel was closed and so
            // need to fail the future right away. If it is not null the handling of the rest
            // will be done in flush0()
            // See https://github.com/netty/netty/issues/2362
            safeSetFailure(promise,
                    newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
        }
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            safeSetFailure(promise, t);
        }
        return;
    }
    // 要写的数据保存到缓存中
    outboundBuffer.addMessage(msg, size, promise);
}
```



其中 AbstractUnsafe 的 outboundBuffer 属性 ChannelOutboundBuffer 存放要写的数据

```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    // 把msg对象封装成缓存节点对象
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    // 更新缓存中的数据总字节大小
    // increment pending bytes after adding message to the unflushed arrays.
    // See https://github.com/netty/netty/issues/1619
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```



分析invokeFlush0方法

最终调到 headcontext 的 flush 方法

```java
public void flush(ChannelHandlerContext ctx) {
    // 真正写数据，并回调 ChannelFutureListener
    unsafe.flush();
}
```



```java
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    // addFlush 的作用就是把 unflushedEntry 链表上的数据转移到 flushedEntry 链表上
    outboundBuffer.addFlush();
    // 真正写数据，并回调 ChannelFutureListener
    flush0();
}
```



```java
protected void flush0() {
    if (inFlush0) {
        // Avoid re-entrance
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    inFlush0 = true;

    // Mark all pending write requests as failure if the channel is inactive.
    if (!isActive()) {
        try {
            // Check if we need to generate the exception at all.
            if (!outboundBuffer.isEmpty()) {
                if (isOpen()) {
                    outboundBuffer.failFlushed(new NotYetConnectedException(), true);
                } else {
                    // Do not trigger channelWritabilityChanged because the channel is closed already.
                    outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause, "flush0()"), false);
                }
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }

    try {
        // 真正写数据，并回调 ChannelFutureListener
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        handleWriteError(t);
    } finally {
        inFlush0 = false;
    }
}
```



```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    // 获取java底层的SocketChannel
    SocketChannel ch = javaChannel();
    // 一次最多可以执行多少次刷出数据到网络操作
    int writeSpinCount = config().getWriteSpinCount();
    do {
        if (in.isEmpty()) {
            // All written so clear OP_WRITE
            // 如果 SocketChannel 在 selector 上注册了 OP_WRITE 事件，那么写完数据之后，取消 SocketChannel 注册的 OP_WRITE 事件
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }
        // 每一次写操作最多可以写多少数据
        // Ensure the pending writes are made of ByteBufs only.
        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        // 初始化写操作 ByteBuffer 数组，重要
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        // 得到 ByteBuffer 数组的长度
        int nioBufferCnt = in.nioBufferCount();

        // Always use nioBuffers() to workaround data-corruption.
        // See https://github.com/netty/netty/issues/2761
        switch (nioBufferCnt) {
            // ByteBuffer数组长度是0
            case 0:
                // We have something else beside ByteBuffers to write so fallback to normal writes.
                // 比如写出的数据是FileRegion类型而不是ByteBuffer类型
                writeSpinCount -= doWrite0(in);
                break;
            case 1: {
                // Only one ByteBuf so use non-gathering write
                // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                // to check if the total size of all the buffers is non-zero.
                // 取得待刷ByteBuffer
                ByteBuffer buffer = nioBuffers[0];
                int attemptedBytes = buffer.remaining();
                // 真正写数据
                final int localWrittenBytes = ch.write(buffer);
                if (localWrittenBytes <= 0) {
                    // 如果SocketChannel没把数据写出去，说socket tcp写缓存已经满了
                    // 那么通过incompleteWrite方法使得SocketChannel向selector注册OP_WRITE，等待socket tcp 写缓存可用
                    incompleteWrite(true);
                    return;
                }
                // 根据试图写入的数据量attemptedBytes，实际写入的数据量localWrittenBytes动态修改单次最大容许写入数据量maxBytesPerGatheringWrite
                adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                // 回调 ChannelFutureListener
                // 根据实际的写入数据量更新ChannelOutboundBuffer的缓存状态
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
            // 如果一次需要写出多个ByteBuffer
            default: {
                // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
                // to check if the total size of all the buffers is non-zero.
                // We limit the max amount to int above so cast is safe
                // 算出多个待写出ByteBuffer的大小
                long attemptedBytes = in.nioBufferSize();
                // SocketChannel写出ByteBuffers到网络，返回写出的数据量
                final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                        maxBytesPerGatheringWrite);
                in.removeBytes(localWrittenBytes);
                // writeSpinCount减去1
                --writeSpinCount;
                break;
            }
        }
    } while (writeSpinCount > 0);
    // setOpWrite 用来表示是不是需要在selector上注册OP_WRITE事件
    incompleteWrite(writeSpinCount < 0);
}
```



```java
public ByteBuffer[] nioBuffers(int maxCount, long maxBytes) {
    assert maxCount > 0;
    assert maxBytes > 0;
    long nioBufferSize = 0;
    int nioBufferCount = 0;
    final InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    // 默认返回一个长度为1024的ByteBuffer数组
    ByteBuffer[] nioBuffers = NIO_BUFFERS.get(threadLocalMap);
    Entry entry = flushedEntry;
    while (isFlushedEntry(entry) && entry.msg instanceof ByteBuf) {
        if (!entry.cancelled) {
            ByteBuf buf = (ByteBuf) entry.msg;
            final int readerIndex = buf.readerIndex();
            // 计算当前缓存节点的大小
            final int readableBytes = buf.writerIndex() - readerIndex;

            if (readableBytes > 0) {
                // 在nioBufferCount！=0 的情况下，如果最大容许写出的数据量小于本次待转化为ByteBuffer的数据量加上已经转化成ByteBuffer的数据量，那么就不再继续做ByteBuffer转化了
                if (maxBytes - readableBytes < nioBufferSize && nioBufferCount != 0) {
                    // If the nioBufferSize + readableBytes will overflow maxBytes, and there is at least one entry
                    // we stop populate the ByteBuffer array. This is done for 2 reasons:
                    // 1. bsd/osx don't allow to write more bytes then Integer.MAX_VALUE with one writev(...) call
                    // and so will return 'EINVAL', which will raise an IOException. On Linux it may work depending
                    // on the architecture and kernel but to be safe we also enforce the limit here.
                    // 2. There is no sense in putting more data in the array than is likely to be accepted by the
                    // OS.
                    //
                    // See also:
                    // - https://www.freebsd.org/cgi/man.cgi?query=write&sektion=2
                    // - https://linux.die.net//man/2/writev
                    break;
                }
                // 更新已经转化成ByteBuffer的缓存节点的总字节大小
                nioBufferSize += readableBytes;
                int count = entry.count;
                if (count == -1) {
                    //noinspection ConstantValueVariableUse
                    entry.count = count = buf.nioBufferCount();
                }
                // nioBufferCount表示已经转化出的ByteBuffer的个数，因为在目前netty这个版本中nioBuffers.length的初始值是等于maxCount的，
                // 所以expandNioBufferArray没有机会得到执行
                int neededSpace = min(maxCount, nioBufferCount + count);
                if (neededSpace > nioBuffers.length) {
                    nioBuffers = expandNioBufferArray(nioBuffers, neededSpace, nioBufferCount);
                    NIO_BUFFERS.set(threadLocalMap, nioBuffers);
                }
                if (count == 1) {
                    ByteBuffer nioBuf = entry.buf;
                    if (nioBuf == null) {
                        // cache ByteBuffer as it may need to create a new ByteBuffer instance if its a
                        // 返回ByteBuf绑定的ByteBuffer(position=readerIndex， limit = readerIndex+readableBytes)的副本
                        entry.buf = nioBuf = buf.internalNioBuffer(readerIndex, readableBytes);
                    }
                    // 设置nioBuffers[nioBufferCount]为当前转化出的ByteBuffer
                    // 同时更新已经转化出的ByteBuffer的数量
                    nioBuffers[nioBufferCount++] = nioBuf;
                } else {
                    // The code exists in an extra method to ensure the method is not too big to inline as this
                    // branch is not very likely to get hit very frequently.
                    nioBufferCount = nioBuffers(entry, buf, nioBuffers, nioBufferCount, maxCount);
                }
                // 如果转化的ByeBuffer的个数已经大于等于了maxCount那么结束转化
                if (nioBufferCount >= maxCount) {
                    break;
                }
            }
        }
        entry = entry.next;
    }
    // 记录本次转化的ByteBuffer的个数
    this.nioBufferCount = nioBufferCount;
    // 记录本次转化出的ByteBuffer的总大小
    this.nioBufferSize = nioBufferSize;

    return nioBuffers;
}
```



```java
public void removeBytes(long writtenBytes) {
    // 如果localWrittenBytes等于attemptedBytes那么把对应的entry从缓存节点中删除
    // 如果localWrittenBytes不等于attemptedBytes说明本次写入没有把一个msg中包含的数据完整的写到网络上，
    // 那么需要更新msg对应的ByteBuf读写指针的位置，等待下次继续执行
    for (;;) {
        Object msg = current();
        if (!(msg instanceof ByteBuf)) {
            assert writtenBytes == 0;
            break;
        }

        final ByteBuf buf = (ByteBuf) msg;
        // 获取缓存数据的对应的ByteBuf的readerIndex
        final int readerIndex = buf.readerIndex();
        // 获取ByteBuf保存的缓存数据的大小
        final int readableBytes = buf.writerIndex() - readerIndex;
        // 如果写出到网络上的数据量大于本节点的缓存数据大小
        // 那么意味着本缓存节点entry已经被全部刷到网络上，所以调用remove()，把entry从缓存链上删除
        if (readableBytes <= writtenBytes) {
            if (writtenBytes != 0) {
                // 如果用户设置了写入进度通知的功能，progress通知用户写入进度
                progress(readableBytes);
                // 更新写入的数据量
                writtenBytes -= readableBytes;
            }
            // 把entry从缓存链上删除, 并回调 ChannelFutureListener
            remove();
        } else { // readableBytes > writtenBytes
            // 当前缓存节点的数据量大于writtenBytes
            // 那么需要更新entry对应ByteBuf的readIndex，这时缓存节点entry还是继续保留在缓存链上，等待下一次继续被写出
            if (writtenBytes != 0) {
                buf.readerIndex(readerIndex + (int) writtenBytes);
                progress(writtenBytes);
            }
            break;
        }
    }
    clearNioBuffers();
}
```



```java
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;
    // 把entry从缓存链上删除,
    removeEntry(e);

    if (!e.cancelled) {
        // only release message, notify and decrement if it was not canceled before.
        ReferenceCountUtil.safeRelease(msg);
        // 回调 ChannelFutureListener
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true);
    }

    // recycle the entry
    e.recycle();

    return true;
}
```
