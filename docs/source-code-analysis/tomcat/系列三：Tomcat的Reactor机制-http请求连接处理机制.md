
# Tomcat的Reactor机制-http请求连接处理机制

[TOC]



## NioEndPoint 初始化

Http11NioProtocol.init->NioEndPoint.init

org.apache.tomcat.util.net.AbstractEndpoint#init

```java
public void init() throws Exception {
    if (bindOnInit) {
        // 绑定端口号
        bind();
        bindState = BindState.BOUND_ON_INIT;
    }
    if (this.domain != null) {
        // Register endpoint (as ThreadPool - historical name)
        oname = new ObjectName(domain + ":type=ThreadPool,name=\"" + getName() + "\"");
        Registry.getRegistry(null, null).registerComponent(this, oname, null);

        ObjectName socketPropertiesOname = new ObjectName(domain +
                ":type=ThreadPool,name=\"" + getName() + "\",subType=SocketProperties");
        socketProperties.setObjectName(socketPropertiesOname);
        Registry.getRegistry(null, null).registerComponent(socketProperties, socketPropertiesOname, null);

        for (SSLHostConfig sslHostConfig : findSslHostConfigs()) {
            registerJmx(sslHostConfig);
        }
    }
}
```

org.apache.tomcat.util.net.NioEndpoint#bind

```java
public void bind() throws Exception {

    if (!getUseInheritedChannel()) {
        // 绑定端口号
        serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));
        serverSock.socket().bind(addr,getAcceptCount());
    } else {
        // Retrieve the channel provided by the OS
        Channel ic = System.inheritedChannel();
        if (ic instanceof ServerSocketChannel) {
            serverSock = (ServerSocketChannel) ic;
        }
        if (serverSock == null) {
            throw new IllegalArgumentException(sm.getString("endpoint.init.bind.inherited"));
        }
    }
    serverSock.configureBlocking(true); //mimic APR behavior

    // Initialize thread count defaults for acceptor, poller
    if (acceptorThreadCount == 0) {
        // FIXME: Doesn't seem to work that well with multiple accept threads
        acceptorThreadCount = 1;
    }
    if (pollerThreadCount <= 0) {
        //minimum one poller thread
        pollerThreadCount = 1;
    }
    setStopLatch(new CountDownLatch(pollerThreadCount));

    // Initialize SSL if needed
    initialiseSsl();

    selectorPool.open();
}
```



Http11NioProtocol.start->NioEndPoint.start

org.apache.tomcat.util.net.NioEndpoint#startInternal

```java
public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getBufferPool());

        // Create worker collection
        // 创建【工作线程池】，Tomcat 自己包装了一下 ThreadPoolExecutor，
        // 1. 为了在创建线程池以后，先启动 corePoolSize 个线程
        // 2. 自己管理线程池的增长方式（默认 corePoolSize 10, maxPoolSize 200）
        if ( getExecutor() == null ) {
            createExecutor();
        }

        // 设置一个栅栏（tomcat 自定义了类 LimitLatch），控制最大的连接数，默认是 10000
        initializeConnectionLatch();

        // Start poller threads
        // 开启 poller 线程
        // 还记得之前 init 的时候，默认地设置了 poller 的数量为 2，所以这里启动 2 个 poller 线程
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }

        // 开启 acceptor 线程，和开启 poller 线程组差不多。
        // init 的时候，默认地，acceptor 的线程数是 1
        startAcceptorThreads();
    }
}
```



## Acceptor线程阻塞接收连接请求

### 创建

org.apache.tomcat.util.net.AbstractEndpoint#startAcceptorThreads

```java
protected final void startAcceptorThreads() {
    int count = getAcceptorThreadCount();
    acceptors = new Acceptor[count];

    for (int i = 0; i < count; i++) {
        acceptors[i] = createAcceptor();
        String threadName = getName() + "-Acceptor-" + i;
        acceptors[i].setThreadName(threadName);
        Thread t = new Thread(acceptors[i], threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }
}
```

org.apache.tomcat.util.net.NioEndpoint#createAcceptor

```java
protected AbstractEndpoint.Acceptor createAcceptor() {
    return new Acceptor();
}
```

### 接收连接请求

```java
protected class Acceptor extends AbstractEndpoint.Acceptor {

    @Override
    public void run() {

        int errorDelay = 0;

        // Loop until we receive a shutdown command
        // 只要 endpoint 处于 running，这里就一直循环
        while (running) {

            // Loop if endpoint is paused
            // 如果 endpoint 处于 pause 状态，这边 Acceptor 用一个 while 循环将自己也挂起
            while (paused && running) {
                state = AcceptorState.PAUSED;
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    // Ignore
                }
            }

            // endpoint 结束了，Acceptor 自然也要结束嘛
            if (!running) {
                break;
            }
            state = AcceptorState.RUNNING;

            try {
                //if we have reached max connections, wait
                // 如果此时达到了最大连接数(之前我们说过，默认是10000)，就等待
                countUpOrAwaitConnection();

                SocketChannel socket = null;
                try {
                    // Accept the next incoming connection from the server
                    // socket
                    // 这里就是接收下一个进来的 SocketChannel
                    // 之前我们设置了 ServerSocketChannel 为阻塞模式，所以这边的 accept 是阻塞的
                    socket = serverSock.accept();
                } catch (IOException ioe) {
                    // We didn't get a socket
                    countDownConnection();
                    if (running) {
                        // Introduce delay if necessary
                        errorDelay = handleExceptionWithDelay(errorDelay);
                        // re-throw
                        throw ioe;
                    } else {
                        break;
                    }
                }
                // Successful accept, reset the error delay
                // accept 成功，将 errorDelay 设置为 0
                errorDelay = 0;

                // Configure the socket
                if (running && !paused) {
                    // setSocketOptions() will hand the socket off to
                    // an appropriate processor if successful
                    // setSocketOptions() 是这里的关键方法，也就是说前面千辛万苦都是为了能到这里进行处理
                    if (!setSocketOptions(socket)) {
                        // 如果上面的方法返回 false，关闭 SocketChannel
                        closeSocket(socket);
                    }
                } else {
                    // 由于 endpoint 不 running 了，或者处于 pause 了，将此 SocketChannel 关闭
                    closeSocket(socket);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("endpoint.accept.fail"), t);
            }
        }
        state = AcceptorState.ENDED;
    }
```

org.apache.tomcat.util.net.NioEndpoint#setSocketOptions

```java
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        //disable blocking, APR style, we are gonna be polling it
        // 设置该 SocketChannel 为非阻塞模式
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        // 设置 socket 的一些属性
        socketProperties.setProperties(sock);

        // 还记得 startInternal 的时候，说过了 nioChannels 是缓存用的。
        NioChannel channel = nioChannels.pop();
        if (channel == null) {
            // 主要是创建读和写的两个 buffer，默认地，读和写 buffer 都是 8192 字节，8k
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            if (isSSLEnabled()) {
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            channel.reset();
        }
        // getPoller0() 会选取所有 poller 中的一个 poller
        // 我们只需要明白，此时，往 poller 中注册了一个 NioChannel 实例，
        // 此实例包含客户端过来的 SocketChannel 和一个 SocketBufferHandler 实例。
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error("",t);
        } catch (Throwable tt) {
            ExceptionUtils.handleThrowable(tt);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}
```

### 向Poller线程的队列中加入任务

org.apache.tomcat.util.net.NioEndpoint.Poller#register

```java
public void register(final NioChannel socket) {
    socket.setPoller(this);
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getSocketProperties().getSoTimeout());
    ka.setWriteTimeout(getSocketProperties().getSoTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    // 添加 event 到 poller 中
    addEvent(r);
}
```

```java
private void addEvent(PollerEvent event) {
    events.offer(event);
    if ( wakeupCounter.incrementAndGet() == 0 ) selector.wakeup();
}
```

## Poller线程注册socket的read事件以及长连接超时处理

### 创建

org.apache.tomcat.util.net.NioEndpoint#startInternal

```java
pollers = new Poller[getPollerThreadCount()];
for (int i=0; i<pollers.length; i++) {
    pollers[i] = new Poller();
    Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
    pollerThread.setPriority(threadPriority);
    pollerThread.setDaemon(true);
    pollerThread.start();
}
```

```java
public Poller() throws IOException {
    // 每个 poller 开启一个 Selector
    this.selector = Selector.open();
}
```

### 注册socket的read事件

```java
public void run() {
    // Loop until destroy() is called
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
                // 执行 events 队列中每个 event 的 run() 方法, 注册监听 socket 的 read 事件
                hasEvents = events();
                // wakeupCounter 的初始值为 0，这里设置为 -1
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    keyCount = selector.selectNow();
                } else {
                    // timeout 默认值 1 秒
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
            }
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error("",x);
            continue;
        }
        //either we timed out or we woke up, process events first
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        // 如果刚刚 select 有返回 ready keys，进行处理
        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                // ※※※※※ 处理 ready key ※※※※※
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        // 长连接关闭请求
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
```

org.apache.tomcat.util.net.NioEndpoint.Poller#events

```java
public boolean events() {
    boolean result = false;

    PollerEvent pe = null;
    for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
        result = true;
        try {
            // 逐个执行 event.run()
            pe.run();
            // 该 PollerEvent 还得给以后用，这里 reset 一下(还是之前说过的缓存)
            pe.reset();
            if (running && !paused) {
                eventCache.push(pe);
            }
        } catch ( Throwable x ) {
            log.error("",x);
        }
    }

    return result;
}
```

org.apache.tomcat.util.net.NioEndpoint.PollerEvent#run

```java
public void run() {
    // 对于新来的连接，interestOps == OP_REGISTER
    if (interestOps == OP_REGISTER) {
        try {
            // 这步很关键！！！
            // 将这个新连接 SocketChannel 注册到该 poller 的 Selector 中，
            // 设置监听 OP_READ 事件，
            // 将 socketWrapper 设置为 attachment 进行传递
            socket.getIOChannel().register(
                    socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
        } catch (Exception x) {
            log.error(sm.getString("endpoint.nio.registerFail"), x);
        }
    } else {
        final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
        try {
            if (key == null) {
                // The key was cancelled (e.g. due to socket closure)
                // and removed from the selector while it was being
                // processed. Count down the connections at this point
                // since it won't have been counted down when the socket
                // closed.
                socket.socketWrapper.getEndpoint().countDownConnection();
                ((NioSocketWrapper) socket.socketWrapper).closed = true;
            } else {
                final NioSocketWrapper socketWrapper = (NioSocketWrapper) key.attachment();
                if (socketWrapper != null) {
                    //we are registering the key to start with, reset the fairness counter.
                    int ops = key.interestOps() | interestOps;
                    socketWrapper.interestOps(ops);
                    key.interestOps(ops);
                } else {
                    socket.getPoller().cancelledKey(key);
                }
            }
        } catch (CancelledKeyException ckx) {
            try {
                socket.getPoller().cancelledKey(key);
            } catch (Exception ignore) {}
        }
    }
}
```

### 处理请求

org.apache.tomcat.util.net.NioEndpoint.Poller#processKey

```java
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                // 忽略 sendfile
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    // unregister 相应的 interest set，
                    // 如接下来是处理 SocketChannel 进来的数据，那么就不再监听该 channel 的 OP_READ 事件
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // Read goes before write
                    if (sk.isReadable()) {
                        // 处理读
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    if (!closeSocket && sk.isWritable()) {
                        // 处理写
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
}
```

org.apache.tomcat.util.net.AbstractEndpoint#processSocket

```java
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
        SocketEvent event, boolean dispatch) {
    try {
        if (socketWrapper == null) {
            return false;
        }
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
            // 创建一个 SocketProcessor 的实例
            sc = createSocketProcessor(socketWrapper, event);
        } else {
            sc.reset(socketWrapper, event);
        }
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            // 将任务放到之前建立的 worker 线程池中执行
            executor.execute(sc);
        } else {
            // ps: 如果 dispatch 为 false，那么就当前线程自己执行
            sc.run();
        }
    } catch (RejectedExecutionException ree) {
        getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
        return false;
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        // This means we got an OOM or similar creating a thread, or that
        // the pool and its queue are full
        getLog().error(sm.getString("endpoint.process.fail"), t);
        return false;
    }
    return true;
}
```

org.apache.tomcat.util.net.NioEndpoint#createSocketProcessor

```java
protected SocketProcessorBase<NioChannel> createSocketProcessor(
        SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
    return new SocketProcessor(socketWrapper, event);
}
```

### 长连接超时关闭连接

org.apache.tomcat.util.net.NioEndpoint.Poller#timeout

```java
protected void timeout(int keyCount, boolean hasEvents) {
    long now = System.currentTimeMillis();
    // This method is called on every loop of the Poller. Don't process
    // timeouts on every loop of the Poller since that would create too
    // much load and timeouts can afford to wait a few seconds.
    // However, do process timeouts if any of the following are true:
    // - the selector simply timed out (suggests there isn't much load)
    // - the nextExpiration time has passed
    // - the server socket is being closed
    if (nextExpiration > 0 && (keyCount > 0 || hasEvents) && (now < nextExpiration) && !close) {
        return;
    }
    //timeout
    int keycount = 0;
    try {
        for (SelectionKey key : selector.keys()) {
            keycount++;
            try {
                NioSocketWrapper ka = (NioSocketWrapper) key.attachment();
                if ( ka == null ) {
                    cancelledKey(key); //we don't support any keys without attachments
                } else if (close) {
                    key.interestOps(0);
                    ka.interestOps(0); //avoid duplicate stop calls
                    processKey(key,ka);
                } else if ((ka.interestOps()&SelectionKey.OP_READ) == SelectionKey.OP_READ ||
                          (ka.interestOps()&SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE) {
                    boolean isTimedOut = false;
                    // Check for read timeout
                    if ((ka.interestOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
                        long delta = now - ka.getLastRead();
                        long timeout = ka.getReadTimeout();
                        isTimedOut = timeout > 0 && delta > timeout;
                    }
                    // Check for write timeout
                    if (!isTimedOut && (ka.interestOps() & SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE) {
                        long delta = now - ka.getLastWrite();
                        long timeout = ka.getWriteTimeout();
                        isTimedOut = timeout > 0 && delta > timeout;
                    }
                    if (isTimedOut) {
                        // 长连接关闭请求
                        key.interestOps(0);
                        ka.interestOps(0); //avoid duplicate timeout calls
                        ka.setError(new SocketTimeoutException());
                        // 向线程池中
                        if (!processSocket(ka, SocketEvent.ERROR, true)) {
                            cancelledKey(key);
                        }
                    }
                }
            }catch ( CancelledKeyException ckx ) {
                cancelledKey(key);
            }
        }//for
    } catch (ConcurrentModificationException cme) {
        // See https://bz.apache.org/bugzilla/show_bug.cgi?id=57943
        log.warn(sm.getString("endpoint.nio.timeoutCme"), cme);
    }
    long prevExp = nextExpiration; //for logging purposes only
    nextExpiration = System.currentTimeMillis() +
            socketProperties.getTimeoutInterval();
    if (log.isTraceEnabled()) {
        log.trace("timeout completed: keys processed=" + keycount +
                "; now=" + now + "; nextExpiration=" + prevExp +
                "; keyCount=" + keyCount + "; hasEvents=" + hasEvents +
                "; eval=" + ((now < prevExp) && (keyCount>0 || hasEvents) && (!close) ));
    }

}
```

org.apache.tomcat.util.net.NioEndpoint.Poller#timeout

```java
protected void timeout(int keyCount, boolean hasEvents) {
    long now = System.currentTimeMillis();
    // This method is called on every loop of the Poller. Don't process
    // timeouts on every loop of the Poller since that would create too
    // much load and timeouts can afford to wait a few seconds.
    // However, do process timeouts if any of the following are true:
    // - the selector simply timed out (suggests there isn't much load)
    // - the nextExpiration time has passed
    // - the server socket is being closed
    if (nextExpiration > 0 && (keyCount > 0 || hasEvents) && (now < nextExpiration) && !close) {
        return;
    }
    //timeout
    int keycount = 0;
    try {
        for (SelectionKey key : selector.keys()) {
            keycount++;
            try {
                NioSocketWrapper ka = (NioSocketWrapper) key.attachment();
                if ( ka == null ) {
                    cancelledKey(key); //we don't support any keys without attachments
                } else if (close) {
                    key.interestOps(0);
                    ka.interestOps(0); //avoid duplicate stop calls
                    processKey(key,ka);
                } else if ((ka.interestOps()&SelectionKey.OP_READ) == SelectionKey.OP_READ ||
                          (ka.interestOps()&SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE) {
                    boolean isTimedOut = false;
                    // Check for read timeout
                    if ((ka.interestOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
                        long delta = now - ka.getLastRead();
                        long timeout = ka.getReadTimeout();
                        isTimedOut = timeout > 0 && delta > timeout;
                    }
                    // Check for write timeout
                    if (!isTimedOut && (ka.interestOps() & SelectionKey.OP_WRITE) == SelectionKey.OP_WRITE) {
                        long delta = now - ka.getLastWrite();
                        long timeout = ka.getWriteTimeout();
                        isTimedOut = timeout > 0 && delta > timeout;
                    }
                    if (isTimedOut) {
                        // 长连接关闭请求
                        key.interestOps(0);
                        ka.interestOps(0); //avoid duplicate timeout calls
                        ka.setError(new SocketTimeoutException());
                        // 向线程池中加入取消socket的
                        if (!processSocket(ka, SocketEvent.ERROR, true)) {
                            cancelledKey(key);
                        }
                    }
                }
            }catch ( CancelledKeyException ckx ) {
                cancelledKey(key);
            }
        }//for
    } catch (ConcurrentModificationException cme) {
        // See https://bz.apache.org/bugzilla/show_bug.cgi?id=57943
        log.warn(sm.getString("endpoint.nio.timeoutCme"), cme);
    }
    long prevExp = nextExpiration; //for logging purposes only
    nextExpiration = System.currentTimeMillis() +
            socketProperties.getTimeoutInterval();
    if (log.isTraceEnabled()) {
        log.trace("timeout completed: keys processed=" + keycount +
                "; now=" + now + "; nextExpiration=" + prevExp +
                "; keyCount=" + keyCount + "; hasEvents=" + hasEvents +
                "; eval=" + ((now < prevExp) && (keyCount>0 || hasEvents) && (!close) ));
    }

}
```

org.apache.tomcat.util.net.NioEndpoint.SocketProcessor#doRun

```java
protected void doRun() {
    NioChannel socket = socketWrapper.getSocket();
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

    try {
        int handshake = -1;

        try {
            if (key != null) {
                if (socket.isHandshakeComplete()) {
                    // No TLS handshaking required. Let the handler
                    // process this socket / event combination.
                    handshake = 0;
                } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                        event == SocketEvent.ERROR) {
                    // Unable to complete the TLS handshake. Treat it as
                    // if the handshake failed.
                    handshake = -1;
                } 
/*省略*/
        } else if (handshake == -1 ) {
            getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL);
            // 关闭socket
            close(socket, key);
        } else if (handshake == SelectionKey.OP_READ){
            socketWrapper.registerReadInterest();
        } else if (handshake == SelectionKey.OP_WRITE){
            socketWrapper.registerWriteInterest();
        }
    } catch (CancelledKeyException cx) {
        socket.getPoller().cancelledKey(key);
    } catch (VirtualMachineError vme) {
        ExceptionUtils.handleThrowable(vme);
    } catch (Throwable t) {
        log.error("", t);
        socket.getPoller().cancelledKey(key);
    } finally {
        socketWrapper = null;
        event = null;
        //return to cache
        if (running && !paused) {
            processorCache.push(this);
        }
    }
}
```

org.apache.tomcat.util.net.NioEndpoint#close

```java
private void close(NioChannel socket, SelectionKey key) {
    try {
        if (socket.getPoller().cancelledKey(key) != null) {
            // SocketWrapper (attachment) was removed from the
            // key - recycle the key. This can only happen once
            // per attempted closure so it is used to determine
            // whether or not to return the key to the cache.
            // We do NOT want to do this more than once - see BZ
            // 57340 / 57943.
            if (log.isDebugEnabled()) {
                log.debug("Socket: [" + socket + "] closed");
            }
            if (running && !paused) {
                if (!nioChannels.push(socket)) {
                    socket.free();
                }
            }
        }
    } catch (Exception x) {
        log.error("",x);
    }
}
```

org.apache.tomcat.util.net.NioEndpoint.Poller#cancelledKey

```java
public NioSocketWrapper cancelledKey(SelectionKey key) {
    NioSocketWrapper ka = null;
    try {
        if ( key == null ) return null;//nothing to do
        ka = (NioSocketWrapper) key.attach(null);
        if (ka != null) {
            // If attachment is non-null then there may be a current
            // connection with an associated processor.
            getHandler().release(ka);
        }
        if (key.isValid()) key.cancel();
        // If it is available, close the NioChannel first which should
        // in turn close the underlying SocketChannel. The NioChannel
        // needs to be closed first, if available, to ensure that TLS
        // connections are shut down cleanly.
        if (ka != null) {
            try {
                // 关闭socket
                ka.getSocket().close(true);
            } catch (Exception e){
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString(
                            "endpoint.debug.socketCloseFail"), e);
                }
            }
        }
        // The SocketChannel is also available via the SelectionKey. If
        // it hasn't been closed in the block above, close it now.
        if (key.channel().isOpen()) {
            try {
                key.channel().close();
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString(
                            "endpoint.debug.channelCloseFail"), e);
                }
            }
        }
        try {
            if (ka != null && ka.getSendfileData() != null
                    && ka.getSendfileData().fchannel != null
                    && ka.getSendfileData().fchannel.isOpen()) {
                ka.getSendfileData().fchannel.close();
            }
        } catch (Exception ignore) {
        }
        if (ka != null) {
            countDownConnection();
            ka.closed = true;
        }
    } catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        if (log.isDebugEnabled()) log.error("",e);
    }
    return ka;
}
```

## 处理请求的线程池

### 创建

org.apache.tomcat.util.net.NioEndpoint#startInternal

```java
if ( getExecutor() == null ) {
    createExecutor();
}
```

org.apache.tomcat.util.net.AbstractEndpoint#createExecutor

```java
public void createExecutor() {
    internalExecutor = true;
    TaskQueue taskqueue = new TaskQueue();
    TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
    executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
    taskqueue.setParent( (ThreadPoolExecutor) executor);
}
```

线程池重写了TaskQueue，其中TaskQueue extends LinkedBlockingQueue<Runnable>

org.apache.tomcat.util.threads.TaskQueue#offer

```java
public boolean offer(Runnable o) {
  //we can't do any checks
    if (parent==null) return super.offer(o);
    //we are maxed out on threads, simply queue the object
    if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
    //we have idle threads, just add it to the queue
    if (parent.getSubmittedCount()<=(parent.getPoolSize())) return super.offer(o);
    //if we have less threads than maximum force creation of a new thread
    if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
    //if we reached here, we need to add it to the queue
    return super.offer(o);
}
```

最终tomcat的请求提交步骤变为：

**使线程数先达到最大线程数, 之后再放入到阻塞队列中**

### 任务真正执行

org.apache.tomcat.util.net.NioEndpoint.SocketProcessor#doRun

```java
protected void doRun() {
    NioChannel socket = socketWrapper.getSocket();
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

    try {
        int handshake = -1;

        try {
            if (key != null) {
                if (socket.isHandshakeComplete()) {
                    // No TLS handshaking required. Let the handler
                    // process this socket / event combination.
                    handshake = 0;
                } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                        event == SocketEvent.ERROR) {
                    // Unable to complete the TLS handshake. Treat it as
                    // if the handshake failed.
                    handshake = -1;
                } else {
                    handshake = socket.handshake(key.isReadable(), key.isWritable());
                    // The handshake process reads/writes from/to the
                    // socket. status may therefore be OPEN_WRITE once
                    // the handshake completes. However, the handshake
                    // happens when the socket is opened so the status
                    // must always be OPEN_READ after it completes. It
                    // is OK to always set this as it is only used if
                    // the handshake completes.
                    event = SocketEvent.OPEN_READ;
                }
            }
        } catch (IOException x) {
            handshake = -1;
            if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
        } catch (CancelledKeyException ckx) {
            handshake = -1;
        }
        if (handshake == 0) {
            SocketState state = SocketState.OPEN;
            // Process the request from this socket
            if (event == null) {
                // 处理请求
                state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
            } else {
                state = getHandler().process(socketWrapper, event);
            }
            if (state == SocketState.CLOSED) {
                close(socket, key);
            }
        } else if (handshake == -1 ) {
            getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL);
            // 关闭连接
            close(socket, key);
        } else if (handshake == SelectionKey.OP_READ){
            socketWrapper.registerReadInterest();
        } else if (handshake == SelectionKey.OP_WRITE){
            socketWrapper.registerWriteInterest();
        }
    } catch (CancelledKeyException cx) {
        socket.getPoller().cancelledKey(key);
    } catch (VirtualMachineError vme) {
        ExceptionUtils.handleThrowable(vme);
    } catch (Throwable t) {
        log.error("", t);
        socket.getPoller().cancelledKey(key);
    } finally {
        socketWrapper = null;
        event = null;
        //return to cache
        if (running && !paused) {
            processorCache.push(this);
        }
    }
}
```

org.apache.coyote.AbstractProtocol.ConnectionHandler#process

```java
public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
    if (getLog().isDebugEnabled()) {
        getLog().debug(sm.getString("abstractConnectionHandler.process",
                wrapper.getSocket(), status));
    }
    if (wrapper == null) {
        // Nothing to do. Socket has been closed.
        return SocketState.CLOSED;
    }

    S socket = wrapper.getSocket();

    Processor processor = connections.get(socket);
    if (getLog().isDebugEnabled()) {
        getLog().debug(sm.getString("abstractConnectionHandler.connectionsGet",
                processor, socket));
    }

    // Timeouts are calculated on a dedicated thread and then
    // dispatched. Because of delays in the dispatch process, the
    // timeout may no longer be required. Check here and avoid
    // unnecessary processing.
    if (SocketEvent.TIMEOUT == status &&
            (processor == null ||
            !processor.isAsync() && !processor.isUpgrade() ||
            processor.isAsync() && !processor.checkAsyncTimeoutGeneration())) {
        // This is effectively a NO-OP
        return SocketState.OPEN;
    }

    if (processor != null) {
        // Make sure an async timeout doesn't fire
        getProtocol().removeWaitingProcessor(processor);
    } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {
        // Nothing to do. Endpoint requested a close and there is no
        // longer a processor associated with this socket.
        return SocketState.CLOSED;
    }

    ContainerThreadMarker.set();

    try {
        if (processor == null) {
            String negotiatedProtocol = wrapper.getNegotiatedProtocol();
            // OpenSSL typically returns null whereas JSSE typically
            // returns "" when no protocol is negotiated
            if (negotiatedProtocol != null && negotiatedProtocol.length() > 0) {
                UpgradeProtocol upgradeProtocol =
                        getProtocol().getNegotiatedProtocol(negotiatedProtocol);
                if (upgradeProtocol != null) {
                    processor = upgradeProtocol.getProcessor(
                            wrapper, getProtocol().getAdapter());
                } else if (negotiatedProtocol.equals("http/1.1")) {
                    // Explicitly negotiated the default protocol.
                    // Obtain a processor below.
                } else {
                    // TODO:
                    // OpenSSL 1.0.2's ALPN callback doesn't support
                    // failing the handshake with an error if no
                    // protocol can be negotiated. Therefore, we need to
                    // fail the connection here. Once this is fixed,
                    // replace the code below with the commented out
                    // block.
                    if (getLog().isDebugEnabled()) {
                        getLog().debug(sm.getString(
                            "abstractConnectionHandler.negotiatedProcessor.fail",
                            negotiatedProtocol));
                    }
                    return SocketState.CLOSED;
                    /*
                     * To replace the code above once OpenSSL 1.1.0 is
                     * used.
                    // Failed to create processor. This is a bug.
                    throw new IllegalStateException(sm.getString(
                            "abstractConnectionHandler.negotiatedProcessor.fail",
                            negotiatedProtocol));
                    */
                }
            }
        }
        if (processor == null) {
            processor = recycledProcessors.pop();
            if (getLog().isDebugEnabled()) {
                getLog().debug(sm.getString("abstractConnectionHandler.processorPop",
                        processor));
            }
        }
        if (processor == null) {
            // 创建 Http11Processor
            processor = getProtocol().createProcessor();
            register(processor);
        }

        processor.setSslSupport(
                wrapper.getSslSupport(getProtocol().getClientCertProvider()));

        // Associate the processor with the connection
        connections.put(socket, processor);

        SocketState state = SocketState.CLOSED;
        do {
            // Http11Processor.process 处理请求
            state = processor.process(wrapper, status);

            if (state == SocketState.UPGRADING) {
                // Get the HTTP upgrade handler
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                // Retrieve leftover input
                ByteBuffer leftOverInput = processor.getLeftoverInput();
                if (upgradeToken == null) {
                    // Assume direct HTTP/2 connection
                    UpgradeProtocol upgradeProtocol = getProtocol().getUpgradeProtocol("h2c");
                    if (upgradeProtocol != null) {
                        processor = upgradeProtocol.getProcessor(
                                wrapper, getProtocol().getAdapter());
                        wrapper.unRead(leftOverInput);
                        // Associate with the processor with the connection
                        connections.put(socket, processor);
                    } else {
                        if (getLog().isDebugEnabled()) {
                            getLog().debug(sm.getString(
                                "abstractConnectionHandler.negotiatedProcessor.fail",
                                "h2c"));
                        }
                        return SocketState.CLOSED;
                    }
                } else {
                    HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                    // Release the Http11 processor to be re-used
                    release(processor);
                    // Create the upgrade processor
                    processor = getProtocol().createUpgradeProcessor(wrapper, upgradeToken);
                    if (getLog().isDebugEnabled()) {
                        getLog().debug(sm.getString("abstractConnectionHandler.upgradeCreate",
                                processor, wrapper));
                    }
                    wrapper.unRead(leftOverInput);
                    // Mark the connection as upgraded
                    wrapper.setUpgraded(true);
                    // Associate with the processor with the connection
                    connections.put(socket, processor);
                    // Initialise the upgrade handler (which may trigger
                    // some IO using the new protocol which is why the lines
                    // above are necessary)
                    // This cast should be safe. If it fails the error
                    // handling for the surrounding try/catch will deal with
                    // it.
                    if (upgradeToken.getInstanceManager() == null) {
                        httpUpgradeHandler.init((WebConnection) processor);
                    } else {
                        ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                        try {
                            httpUpgradeHandler.init((WebConnection) processor);
                        } finally {
                            upgradeToken.getContextBind().unbind(false, oldCL);
                        }
                    }
                }
            }
        } while ( state == SocketState.UPGRADING);

        if (state == SocketState.LONG) {
            // In the middle of processing a request/response. Keep the
            // socket associated with the processor. Exact requirements
            // depend on type of long poll
            longPoll(wrapper, processor);
            if (processor.isAsync()) {
                getProtocol().addWaitingProcessor(processor);
            }
        } else if (state == SocketState.OPEN) {
            // In keep-alive but between requests. OK to recycle
            // processor. Continue to poll for the next request.
            connections.remove(socket);
            release(processor);
            wrapper.registerReadInterest();
        } else if (state == SocketState.SENDFILE) {
            // Sendfile in progress. If it fails, the socket will be
            // closed. If it works, the socket either be added to the
            // poller (or equivalent) to await more data or processed
            // if there are any pipe-lined requests remaining.
        } else if (state == SocketState.UPGRADED) {
            // Don't add sockets back to the poller if this was a
            // non-blocking write otherwise the poller may trigger
            // multiple read events which may lead to thread starvation
            // in the connector. The write() method will add this socket
            // to the poller if necessary.
            if (status != SocketEvent.OPEN_WRITE) {
                longPoll(wrapper, processor);
                getProtocol().addWaitingProcessor(processor);
            }
        } else if (state == SocketState.SUSPENDED) {
            // Don't add sockets back to the poller.
            // The resumeProcessing() method will add this socket
            // to the poller.
        } else {
            // Connection closed. OK to recycle the processor. Upgrade
            // processors are not recycled.
            connections.remove(socket);
            if (processor.isUpgrade()) {
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                InstanceManager instanceManager = upgradeToken.getInstanceManager();
                if (instanceManager == null) {
                    httpUpgradeHandler.destroy();
                } else {
                    ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                    try {
                        httpUpgradeHandler.destroy();
                    } finally {
                        try {
                            instanceManager.destroyInstance(httpUpgradeHandler);
                        } catch (Throwable e) {
                            ExceptionUtils.handleThrowable(e);
                            getLog().error(sm.getString("abstractConnectionHandler.error"), e);
                        }
                        upgradeToken.getContextBind().unbind(false, oldCL);
                    }
                }
            } else {
                release(processor);
            }
        }
        return state;
    } catch(java.net.SocketException e) {
        // SocketExceptions are normal
        getLog().debug(sm.getString(
                "abstractConnectionHandler.socketexception.debug"), e);
    } catch (java.io.IOException e) {
        // IOExceptions are normal
        getLog().debug(sm.getString(
                "abstractConnectionHandler.ioexception.debug"), e);
    } catch (ProtocolException e) {
        // Protocol exceptions normally mean the client sent invalid or
        // incomplete data.
        getLog().debug(sm.getString(
                "abstractConnectionHandler.protocolexception.debug"), e);
    }
    // Future developers: if you discover any other
    // rare-but-nonfatal exceptions, catch them here, and log as
    // above.
    catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        // any other exception or error is odd. Here we log it
        // with "ERROR" level, so it will show up even on
        // less-than-verbose logs.
        getLog().error(sm.getString("abstractConnectionHandler.error"), e);
    } finally {
        ContainerThreadMarker.clear();
    }

    // Make sure socket/processor is removed from the list of current
    // connections
    connections.remove(socket);
    release(processor);
    return SocketState.CLOSED;
}
```

org.apache.coyote.AbstractProcessorLight#process

```java
public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
        throws IOException {

    SocketState state = SocketState.CLOSED;
    Iterator<DispatchType> dispatches = null;
    do {
        if (dispatches != null) {
            DispatchType nextDispatch = dispatches.next();
            state = dispatch(nextDispatch.getSocketStatus());
        } else if (status == SocketEvent.DISCONNECT) {
            // Do nothing here, just wait for it to get recycled
        } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
            state = dispatch(status);
            if (state == SocketState.OPEN) {
                // 处理请求
                state = service(socketWrapper);
            }
        } else if (status == SocketEvent.OPEN_WRITE) {
            // Extra write event likely after async, ignore
            state = SocketState.LONG;
        } else if (status == SocketEvent.OPEN_READ) {
            state = service(socketWrapper);
        } else if (status == SocketEvent.CONNECT_FAIL) {
            logAccess(socketWrapper);
        } else {
            // Default to closing the socket if the SocketEvent passed in
            // is not consistent with the current state of the Processor
            state = SocketState.CLOSED;
        }

        if (getLog().isDebugEnabled()) {
            getLog().debug("Socket: [" + socketWrapper +
                    "], Status in: [" + status +
                    "], State out: [" + state + "]");
        }

        if (state != SocketState.CLOSED && isAsync()) {
            state = asyncPostProcess();
            if (getLog().isDebugEnabled()) {
                getLog().debug("Socket: [" + socketWrapper +
                        "], State after async post processing: [" + state + "]");
            }
        }

        if (dispatches == null || !dispatches.hasNext()) {
            // Only returns non-null iterator if there are
            // dispatches to process.
            dispatches = getIteratorAndClearDispatches();
        }
    } while (state == SocketState.ASYNC_END ||
            dispatches != null && state != SocketState.CLOSED);

    return state;
}
```

之后调用org.apache.coyote.http11.Http11Processor#service方法来处理请求

## 总结

1. Tomcat获取连接的时候使用阻塞获取连接的方式，没有使用reactor模型。防止由于selector.select(selectorTimeout)导致的cpu空转，避免了cpu空转的场景。
2. Tomcat处理长连接超时以及获取read事件的时候使用了reactor模式，非阻塞获取可读的socket，提高请求处理的能力。
3. 请求流程如下：

![image-20220721210422762](assets/image-20220721210422762.png)

