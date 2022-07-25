

# Tomcat的Listener、Filter、Servlet的调用

[TOC]



## 简介

本文只分析析http1.1的请求，tomcat版本：8.4.45

## 请求的处理链路

### 实例化Http11NioProtocol

根据server.xml中的`<connector/>`来实例化Http11NioProtocol

```html
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```

- protocol是应用层协议名，可填参数有HTTP/1.1、org.apache.coyote.http11.Http11NioProtocol、AJP/1.3、org.apache.coyote.ajp.AjpNioProtocol，如果protocol不填，则默认为Http11NioProtocol。

### NioEndPoint获取到socket请求

Acceptor接收到请求封装成一个SocketProcessor扔进线程池Executor后，会调用Processor从操作系统底层读取、过滤字节流，对应用层协议（HTTP/AJP）进行解析封装，生成org.apache.coyote.Request和org.apache.coyote.Response对象。不同的协议有不同的Processor，HTTP/1.1对应Http11Processor，AJP对应AjpProcessor，HTTP/1.2对应StreamProcessor，UpgradeProcessorInternal 和 UpgradeProcessorExternal用于协议升级：

SocketProcessor并不是直接调用的Processor，而是通过org.apache.coyote.AbstractProtocol.ConnectionHandler#process找到一个合适的Processor进行请求处理：根据不同协议创建Http11Processor or AjpProcessor；

详见[系列二：tomcat的Reactor机制/http请求连接处理机制](docs/source-code-analysis/tomcat/tomcat的Reactor机制-http请求连接处理机制.md)


### 根据Http11NioProtocol得到Http11Processor

```java
protected Processor createProcessor() {
        Http11Processor processor = new Http11Processor(getMaxHttpHeaderSize(),
                getAllowHostHeaderMismatch(), getRejectIllegalHeaderName(), getEndpoint(),
                getMaxTrailerSize(), allowedTrailerHeaders, getMaxExtensionSize(),
                getMaxSwallowSize(), httpUpgradeProtocols, getSendReasonPhrase(),
                relaxedPathChars, relaxedQueryChars);
        processor.setAdapter(getAdapter());
        processor.setMaxKeepAliveRequests(getMaxKeepAliveRequests());
        processor.setConnectionUploadTimeout(getConnectionUploadTimeout());
        processor.setDisableUploadTimeout(getDisableUploadTimeout());
        processor.setCompressionMinSize(getCompressionMinSize());
        processor.setCompression(getCompression());
        processor.setNoCompressionUserAgents(getNoCompressionUserAgents());
        processor.setCompressibleMimeTypes(getCompressibleMimeTypes());
        processor.setRestrictedUserAgents(getRestrictedUserAgents());
        processor.setMaxSavePostSize(getMaxSavePostSize());
        processor.setServer(getServer());
        processor.setServerRemoveAppProvidedValues(getServerRemoveAppProvidedValues());
        return processor;
}
```

### Http11Processor解析http请求

```java
public SocketState service(SocketWrapperBase<?> socketWrapper)
  throws IOException {
  RequestInfo rp = request.getRequestProcessor();
  rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

  // Setting up the I/O
  setSocketWrapper(socketWrapper);

  // Flags
  keepAlive = true;
  openSocket = false;
  readComplete = true;
  boolean keptAlive = false;
  SendfileState sendfileState = SendfileState.DONE;

  while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
         sendfileState == SendfileState.DONE && !endpoint.isPaused()) {

    // Parsing the request header
    try {
      if (!inputBuffer.parseRequestLine(keptAlive)) {
        if (inputBuffer.getParsingRequestLinePhase() == -1) {
          return SocketState.UPGRADING;
        } else if (handleIncompleteRequestLineRead()) {
          break;
        }
      }

      if (endpoint.isPaused()) {
        // 503 - Service unavailable
        response.setStatus(503);
        setErrorState(ErrorState.CLOSE_CLEAN, null);
      } else {
        keptAlive = true;
        // Set this every time in case limit has been changed via JMX
        request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
        if (!inputBuffer.parseHeaders()) {
          // We've read part of the request, don't recycle it
          // instead associate it with the socket
          openSocket = true;
          readComplete = false;
          break;
        }
        if (!disableUploadTimeout) {
          socketWrapper.setReadTimeout(connectionUploadTimeout);
        }
      }
    } catch (IOException e) {
      if (log.isDebugEnabled()) {
        log.debug(sm.getString("http11processor.header.parse"), e);
      }
      setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
      break;
    } catch (Throwable t) {
      ExceptionUtils.handleThrowable(t);
      UserDataHelper.Mode logMode = userDataHelper.getNextMode();
      if (logMode != null) {
        String message = sm.getString("http11processor.header.parse");
        switch (logMode) {
          case INFO_THEN_DEBUG:
            message += sm.getString("http11processor.fallToDebug");
            //$FALL-THROUGH$
          case INFO:
            log.info(message, t);
            break;
          case DEBUG:
            log.debug(message, t);
        }
      }
      // 400 - Bad Request
      response.setStatus(400);
      setErrorState(ErrorState.CLOSE_CLEAN, t);
    }

    // Has an upgrade been requested?
    Enumeration<String> connectionValues = request.getMimeHeaders().values("Connection");
    boolean foundUpgrade = false;
    while (connectionValues.hasMoreElements() && !foundUpgrade) {
      String connectionValue = connectionValues.nextElement();
      if (connectionValue != null) {
        foundUpgrade = connectionValue.toLowerCase(Locale.ENGLISH).contains("upgrade");
      }
    }

    if (foundUpgrade) {
      // Check the protocol
      String requestedProtocol = request.getHeader("Upgrade");

      UpgradeProtocol upgradeProtocol = httpUpgradeProtocols.get(requestedProtocol);
      if (upgradeProtocol != null) {
        if (upgradeProtocol.accept(request)) {
          // TODO Figure out how to handle request bodies at this
          // point.
          response.setStatus(HttpServletResponse.SC_SWITCHING_PROTOCOLS);
          response.setHeader("Connection", "Upgrade");
          response.setHeader("Upgrade", requestedProtocol);
          action(ActionCode.CLOSE,  null);
          getAdapter().log(request, response, 0);

          InternalHttpUpgradeHandler upgradeHandler =
            upgradeProtocol.getInternalUpgradeHandler(
            getAdapter(), cloneRequest(request));
          UpgradeToken upgradeToken = new UpgradeToken(upgradeHandler, null, null);
          action(ActionCode.UPGRADE, upgradeToken);
          return SocketState.UPGRADING;
        }
      }
    }

    if (getErrorState().isIoAllowed()) {
      // Setting up filters, and parse some request headers
      rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
      try {
        prepareRequest();
      } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        if (log.isDebugEnabled()) {
          log.debug(sm.getString("http11processor.request.prepare"), t);
        }
        // 500 - Internal Server Error
        response.setStatus(500);
        setErrorState(ErrorState.CLOSE_CLEAN, t);
      }
    }

    if (maxKeepAliveRequests == 1) {
      keepAlive = false;
    } else if (maxKeepAliveRequests > 0 &&
               socketWrapper.decrementKeepAlive() <= 0) {
      keepAlive = false;
    }

    // Process the request in the adapter
    if (getErrorState().isIoAllowed()) {
      try {
        rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
        getAdapter().service(request, response);
        // Handle when the response was committed before a serious
        // error occurred.  Throwing a ServletException should both
        // set the status to 500 and set the errorException.
        // If we fail here, then the response is likely already
        // committed, so we can't try and set headers.
        if(keepAlive && !getErrorState().isError() && !isAsync() &&
           statusDropsConnection(response.getStatus())) {
          setErrorState(ErrorState.CLOSE_CLEAN, null);
        }
      } catch (InterruptedIOException e) {
        setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
      } catch (HeadersTooLargeException e) {
        log.error(sm.getString("http11processor.request.process"), e);
        // The response should not have been committed but check it
        // anyway to be safe
        if (response.isCommitted()) {
          setErrorState(ErrorState.CLOSE_NOW, e);
        } else {
          response.reset();
          response.setStatus(500);
          setErrorState(ErrorState.CLOSE_CLEAN, e);
          response.setHeader("Connection", "close"); // TODO: Remove
        }
      } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error(sm.getString("http11processor.request.process"), t);
        // 500 - Internal Server Error
        response.setStatus(500);
        setErrorState(ErrorState.CLOSE_CLEAN, t);
        getAdapter().log(request, response, 0);
      }
    }

    // Finish the handling of the request
    rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);
    if (!isAsync()) {
      // If this is an async request then the request ends when it has
      // been completed. The AsyncContext is responsible for calling
      // endRequest() in that case.
      endRequest();
    }
    rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);

    // If there was an error, make sure the request is counted as
    // and error, and update the statistics counter
    if (getErrorState().isError()) {
      response.setStatus(500);
    }

    if (!isAsync() || getErrorState().isError()) {
      request.updateCounters();
      if (getErrorState().isIoAllowed()) {
        inputBuffer.nextRequest();
        outputBuffer.nextRequest();
      }
    }

    if (!disableUploadTimeout) {
      int soTimeout = endpoint.getConnectionTimeout();
      if(soTimeout > 0) {
        socketWrapper.setReadTimeout(soTimeout);
      } else {
        socketWrapper.setReadTimeout(0);
      }
    }

    rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE);

    sendfileState = processSendfile(socketWrapper);
  }

  rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

  if (getErrorState().isError() || endpoint.isPaused()) {
    return SocketState.CLOSED;
  } else if (isAsync()) {
    return SocketState.LONG;
  } else if (isUpgrade()) {
    return SocketState.UPGRADING;
  } else {
    if (sendfileState == SendfileState.PENDING) {
      return SocketState.SENDFILE;
    } else {
      if (openSocket) {
        if (readComplete) {
          return SocketState.OPEN;
        } else {
          return SocketState.LONG;
        }
      } else {
        return SocketState.CLOSED;
      }
    }
  }
}
```



### 调用CoyoteAdapter#service处理请求

### 调用Container的Valve来处理请求

### 调用StandardWrapperValve的invoke方法真正执行filter以及servlet里面的逻辑

























## 参考

- https://blog.csdn.net/weixin_41835612/article/details/111401819
