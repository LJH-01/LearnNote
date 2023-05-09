[TOC]



# Mq客户端



## Provider端



### 发送的逻辑

```java
// 选择连接
transport = (ProducerTransport) loadBalance.electTransport(transports,errTransports, message, clusterManager.getDataCenter()
        , topicConfig);

PutMessage putMessage = new PutMessage();
putMessage.getHeader().setAcknowledge(config.getAcknowledge());
putMessage.producerId(transport.getProducerId())
        .messages(messages.toArray(new Message[messages.size()]));
putMessage.queueId(loadBalance.electQueueId(transport,message,clusterManager.getDataCenter(), topicConfig));
Command ack = null;
if(!isAsync) {
    ack = transport.sync(putMessage, (int) sendTimeout);
}else{
    transport.async(putMessage,(int) sendTimeout,callback);
    ack = Command.createBooleanAck(putMessage.getRequestId(),JMQCode.SUCCESS);
}
Header header = ack.getHeader();
// 判断服务不健康
if (!checkSuccess(header, transport)) {
    throw new JMQException(header.getError(), header.getStatus());
}
```



```java
public Command sync(final Channel channel, final Command command, final int timeout) throws JMQException {
    if (channel == null) {
        throw new IllegalArgumentException("The argument channel must not be null");
    }

    if (command == null) {
        throw new IllegalArgumentException("The argument command must not be null");
    }
    int sendTimeout = timeout <= 0 ? config.getSendTimeout() : timeout;
    // 同步调用
    ResponseFuture responseFuture =
            new ResponseFuture(channel, command, sendTimeout, null, null, new CountDownLatch(1));

    futures.put(command.getRequestId(), responseFuture);
    if (logger.isTraceEnabled()) {
        logger.trace("put:" + command.getRequestId() + ", begin:" + responseFuture
                .getBeginTime() + ",timeout:" + responseFuture.getTimeout());
    }
    // 发送数据,应答成功回来或超时会自动释放command
    channel.writeAndFlush(command).addListener(new ResponseListener(responseFuture, futures));

    try {

        // 等待命令返回
        Command response;
        try {
            response = responseFuture.await();
        } catch (InterruptedException e) {
            throw new JMQException(JMQCode.CN_THREAD_INTERRUPTED);
        }
        if (null == response) {
            // 发送请求成功，等待应答超时
            if (responseFuture.isSuccess()) {
                throw new JMQException(JMQCode.CN_REQUEST_TIMEOUT, NetUtil.getAddress(channel.remoteAddress()));
            } else {
                // 发送请求失败
                Throwable cause = responseFuture.getCause();
                if (cause != null) {
                    if (cause instanceof JMQException) {
                        throw (JMQException) cause;
                    }
                    throw new JMQException(JMQCode.CN_REQUEST_ERROR, cause);
                }
                throw new JMQException(JMQCode.CN_REQUEST_ERROR);
            }
        }

        return response;
    } catch (Throwable e) {
        // 出现异常
        command.release();

        futures.remove(command.getRequestId());
        if (e instanceof JMQException) {
            throw (JMQException) e;
        }
        throw new JMQException(JMQCode.CN_REQUEST_ERROR, e);
    }
}
```



















