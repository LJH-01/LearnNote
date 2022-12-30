

# Mysql的JDBC驱动源码解析



[TOC]

本文分析基于mysql-connector-java的5.1.44版本

## 获取数据库连接

### 创建socket连接

调用com.mysql.jdbc.NonRegisteringDriver的connect方法来创建数据库连接

```java
public java.sql.Connection connect(String url, Properties info) throws SQLException {
    if (url == null) {
        throw SQLError.createSQLException(Messages.getString("NonRegisteringDriver.1"), SQLError.SQL_STATE_UNABLE_TO_CONNECT_TO_DATASOURCE, null);
    }

    if (StringUtils.startsWithIgnoreCase(url, LOADBALANCE_URL_PREFIX)) {
        return connectLoadBalanced(url, info);
    } else if (StringUtils.startsWithIgnoreCase(url, REPLICATION_URL_PREFIX)) {
        return connectReplicationConnection(url, info);
    }

    Properties props = null;

    if ((props = parseURL(url, info)) == null) {
        return null;
    }

    if (!"1".equals(props.getProperty(NUM_HOSTS_PROPERTY_KEY))) {
        return connectFailover(url, info);
    }

    try {
        // 真正创建数据库连接
        Connection newConn = com.mysql.jdbc.ConnectionImpl.getInstance(host(props), port(props), props, database(props), url);

        return newConn;
    } catch (SQLException sqlEx) {
        // Don't wrap SQLExceptions, throw
        // them un-changed.
        throw sqlEx;
    } catch (Exception ex) {
        SQLException sqlEx = SQLError.createSQLException(
                Messages.getString("NonRegisteringDriver.17") + ex.toString() + Messages.getString("NonRegisteringDriver.18"),
                SQLError.SQL_STATE_UNABLE_TO_CONNECT_TO_DATASOURCE, null);

        sqlEx.initCause(ex);

        throw sqlEx;
    }
}
```



```java
protected static Connection getInstance(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url)
        throws SQLException {
    if (!Util.isJdbc4()) {
        return new ConnectionImpl(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
    }
    // 反射创建JDBC4Connection
    return (Connection) Util.handleNewInstance(JDBC_4_CONNECTION_CTOR,
            new Object[] { hostToConnectTo, Integer.valueOf(portToConnectTo), info, databaseToConnectTo, url }, null);
}
```

实例化JDBC4Connection，在JDBC4Connection构造方法中创建数据库连接并鉴权

```java
public JDBC4Connection(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {
    super(hostToConnectTo, portToConnectTo, info, databaseToConnectTo, url);
}
```

调用父类ConnectionImpl的构造方法

```java
public ConnectionImpl(String hostToConnectTo, int portToConnectTo, Properties info, String databaseToConnectTo, String url) throws SQLException {

    this.connectionCreationTimeMillis = System.currentTimeMillis();

    if (databaseToConnectTo == null) {
        databaseToConnectTo = "";
    }

    // Stash away for later, used to clone this connection for Statement.cancel and Statement.setQueryTimeout().
    //

    this.origHostToConnectTo = hostToConnectTo;
    this.origPortToConnectTo = portToConnectTo;
    this.origDatabaseToConnectTo = databaseToConnectTo;

    try {
        Blob.class.getMethod("truncate", new Class[] { Long.TYPE });

        this.isRunningOnJDK13 = false;
    } catch (NoSuchMethodException nsme) {
        this.isRunningOnJDK13 = true;
    }

    this.sessionCalendar = new GregorianCalendar();
    this.utcCalendar = new GregorianCalendar();
    this.utcCalendar.setTimeZone(TimeZone.getTimeZone("GMT"));

    //
    // Normally, this code would be in initializeDriverProperties, but we need to do this as early as possible, so we can start logging to the 'correct'
    // place as early as possible...this.log points to 'NullLogger' for every connection at startup to avoid NPEs and the overhead of checking for NULL at
    // every logging call.
    //
    // We will reset this to the configured logger during properties initialization.
    //
    this.log = LogFactory.getLogger(getLogger(), LOGGER_INSTANCE_NAME, getExceptionInterceptor());

    if (NonRegisteringDriver.isHostPropertiesList(hostToConnectTo)) {
        Properties hostSpecificProps = NonRegisteringDriver.expandHostKeyValues(hostToConnectTo);

        Enumeration<?> propertyNames = hostSpecificProps.propertyNames();

        while (propertyNames.hasMoreElements()) {
            String propertyName = propertyNames.nextElement().toString();
            String propertyValue = hostSpecificProps.getProperty(propertyName);

            info.setProperty(propertyName, propertyValue);
        }
    } else {

        if (hostToConnectTo == null) {
            this.host = "localhost";
            this.hostPortPair = this.host + ":" + portToConnectTo;
        } else {
            this.host = hostToConnectTo;

            if (hostToConnectTo.indexOf(":") == -1) {
                this.hostPortPair = this.host + ":" + portToConnectTo;
            } else {
                this.hostPortPair = this.host;
            }
        }
    }

    this.port = portToConnectTo;

    this.database = databaseToConnectTo;
    this.myURL = url;
    this.user = info.getProperty(NonRegisteringDriver.USER_PROPERTY_KEY);
    this.password = info.getProperty(NonRegisteringDriver.PASSWORD_PROPERTY_KEY);

    if ((this.user == null) || this.user.equals("")) {
        this.user = "";
    }

    if (this.password == null) {
        this.password = "";
    }

    this.props = info;

    initializeDriverProperties(info);

    // We store this per-connection, due to static synchronization issues in Java's built-in TimeZone class...
    this.defaultTimeZone = TimeUtil.getDefaultTimeZone(getCacheDefaultTimezone());

    this.isClientTzUTC = !this.defaultTimeZone.useDaylightTime() && this.defaultTimeZone.getRawOffset() == 0;

    if (getUseUsageAdvisor()) {
        this.pointOfOrigin = LogUtils.findCallingClassAndMethod(new Throwable());
    } else {
        this.pointOfOrigin = "";
    }

    try {
        this.dbmd = getMetaData(false, false);
        initializeSafeStatementInterceptors();
        // 真正的创建连接
        createNewIO(false);
        unSafeStatementInterceptors();
    } catch (SQLException ex) {
        cleanup(ex);

        // don't clobber SQL exceptions
        throw ex;
    } catch (Exception ex) {
        cleanup(ex);

        StringBuilder mesg = new StringBuilder(128);

        if (!getParanoid()) {
            mesg.append("Cannot connect to MySQL server on ");
            mesg.append(this.host);
            mesg.append(":");
            mesg.append(this.port);
            mesg.append(".\n\n");
            mesg.append("Make sure that there is a MySQL server ");
            mesg.append("running on the machine/port you are trying ");
            mesg.append("to connect to and that the machine this software is running on ");
            mesg.append("is able to connect to this host/port (i.e. not firewalled). ");
            mesg.append("Also make sure that the server has not been started with the --skip-networking ");
            mesg.append("flag.\n\n");
        } else {
            mesg.append("Unable to connect to database.");
        }

        SQLException sqlEx = SQLError.createSQLException(mesg.toString(), SQLError.SQL_STATE_COMMUNICATION_LINK_FAILURE, getExceptionInterceptor());

        sqlEx.initCause(ex);

        throw sqlEx;
    }

    NonRegisteringDriver.trackConnection(this);
}
```

继续分析createNewIO方法

```java
public void createNewIO(boolean isForReconnect) throws SQLException {
    synchronized (getConnectionMutex()) {
        // Synchronization Not needed for *new* connections, but defintely for connections going through fail-over, since we might get the new connection up
        // and running *enough* to start sending cached or still-open server-side prepared statements over to the backend before we get a chance to
        // re-prepare them...

        Properties mergedProps = exposeAsProperties(this.props);

        if (!getHighAvailability()) {
            connectOneTryOnly(isForReconnect, mergedProps);

            return;
        }

        // 失败重试创建连接
        connectWithRetries(isForReconnect, mergedProps);
    }
}
```

```java
private void connectWithRetries(boolean isForReconnect, Properties mergedProps) throws SQLException {
    double timeout = getInitialTimeout();
    boolean connectionGood = false;

    Exception connectionException = null;

    for (int attemptCount = 0; (attemptCount < getMaxReconnects()) && !connectionGood; attemptCount++) {
        try {
            if (this.io != null) {
                this.io.forceClose();
            }

            // 创建连接
            coreConnect(mergedProps);
            pingInternal(false, 0);

            boolean oldAutoCommit;
            int oldIsolationLevel;
            boolean oldReadOnly;
            String oldCatalog;

            synchronized (getConnectionMutex()) {
                this.connectionId = this.io.getThreadId();
                this.isClosed = false;

                // save state from old connection
                oldAutoCommit = getAutoCommit();
                oldIsolationLevel = this.isolationLevel;
                oldReadOnly = isReadOnly(false);
                oldCatalog = getCatalog();

                this.io.setStatementInterceptors(this.statementInterceptors);
            }

            // Server properties might be different from previous connection, so initialize again...
            initializePropsFromServer();

            if (isForReconnect) {
                // Restore state from old connection
                setAutoCommit(oldAutoCommit);

                if (this.hasIsolationLevels) {
                    setTransactionIsolation(oldIsolationLevel);
                }

                setCatalog(oldCatalog);
                setReadOnly(oldReadOnly);
            }

            connectionGood = true;

            break;
        } catch (Exception EEE) {
            connectionException = EEE;
            connectionGood = false;
        }

        if (connectionGood) {
            break;
        }

        if (attemptCount > 0) {
            try {
                Thread.sleep((long) timeout * 1000);
            } catch (InterruptedException IE) {
                // ignore
            }
        }
    } // end attempts for a single host

    if (!connectionGood) {
        // We've really failed!
        SQLException chainedEx = SQLError.createSQLException(
                Messages.getString("Connection.UnableToConnectWithRetries", new Object[] { Integer.valueOf(getMaxReconnects()) }),
                SQLError.SQL_STATE_UNABLE_TO_CONNECT_TO_DATASOURCE, getExceptionInterceptor());
        chainedEx.initCause(connectionException);

        throw chainedEx;
    }

    if (getParanoid() && !getHighAvailability()) {
        this.password = null;
        this.user = null;
    }

    if (isForReconnect) {
        //
        // Retrieve any 'lost' prepared statements if re-connecting
        //
        Iterator<Statement> statementIter = this.openStatements.iterator();

        //
        // We build a list of these outside the map of open statements, because in the process of re-preparing, we might end up having to close a prepared
        // statement, thus removing it from the map, and generating a ConcurrentModificationException
        //
        Stack<Statement> serverPreparedStatements = null;

        while (statementIter.hasNext()) {
            Statement statementObj = statementIter.next();

            if (statementObj instanceof ServerPreparedStatement) {
                if (serverPreparedStatements == null) {
                    serverPreparedStatements = new Stack<Statement>();
                }

                serverPreparedStatements.add(statementObj);
            }
        }

        if (serverPreparedStatements != null) {
            while (!serverPreparedStatements.isEmpty()) {
                ((ServerPreparedStatement) serverPreparedStatements.pop()).rePrepare();
            }
        }
    }
}
```

```java
private void coreConnect(Properties mergedProps) throws SQLException, IOException {
    int newPort = 3306;
    String newHost = "localhost";

    String protocol = mergedProps.getProperty(NonRegisteringDriver.PROTOCOL_PROPERTY_KEY);

    if (protocol != null) {
        // "new" style URL

        if ("tcp".equalsIgnoreCase(protocol)) {
            newHost = normalizeHost(mergedProps.getProperty(NonRegisteringDriver.HOST_PROPERTY_KEY));
            newPort = parsePortNumber(mergedProps.getProperty(NonRegisteringDriver.PORT_PROPERTY_KEY, "3306"));
        } else if ("pipe".equalsIgnoreCase(protocol)) {
            setSocketFactoryClassName(NamedPipeSocketFactory.class.getName());

            String path = mergedProps.getProperty(NonRegisteringDriver.PATH_PROPERTY_KEY);

            if (path != null) {
                mergedProps.setProperty(NamedPipeSocketFactory.NAMED_PIPE_PROP_NAME, path);
            }
        } else {
            // normalize for all unknown protocols
            newHost = normalizeHost(mergedProps.getProperty(NonRegisteringDriver.HOST_PROPERTY_KEY));
            newPort = parsePortNumber(mergedProps.getProperty(NonRegisteringDriver.PORT_PROPERTY_KEY, "3306"));
        }
    } else {

        String[] parsedHostPortPair = NonRegisteringDriver.parseHostPortPair(this.hostPortPair);
        newHost = parsedHostPortPair[NonRegisteringDriver.HOST_NAME_INDEX];

        newHost = normalizeHost(newHost);

        if (parsedHostPortPair[NonRegisteringDriver.PORT_NUMBER_INDEX] != null) {
            newPort = parsePortNumber(parsedHostPortPair[NonRegisteringDriver.PORT_NUMBER_INDEX]);
        }
    }

    this.port = newPort;
    this.host = newHost;

    // reset max-rows to default value
    this.sessionMaxRows = -1;

    // preconfigure some server variables which are consulted before their initialization from server
    this.serverVariables = new HashMap<String, String>();
    this.serverVariables.put("character_set_server", "utf8");

    // 创建socket连接
    this.io = new MysqlIO(newHost, newPort, mergedProps, getSocketFactoryClassName(), getProxy(), getSocketTimeout(),
            this.largeRowSizeThreshold.getValueAsInt());
    // 使用用户名、密码进行鉴权
    this.io.doHandshake(this.user, this.password, this.database);
    if (versionMeetsMinimum(5, 5, 0)) {
        // error messages are returned according to character_set_results which, at this point, is set from the response packet
        this.errorMessageEncoding = this.io.getEncodingForHandshake();
    }
}
```





```java
public MysqlIO(String host, int port, Properties props, String socketFactoryClassName, MySQLConnection conn, int socketTimeout,
        int useBufferRowSizeThreshold) throws IOException, SQLException {
    this.connection = conn;

    if (this.connection.getEnablePacketDebug()) {
        this.packetDebugRingBuffer = new LinkedList<StringBuilder>();
    }
    this.traceProtocol = this.connection.getTraceProtocol();

    this.useAutoSlowLog = this.connection.getAutoSlowLog();

    this.useBufferRowSizeThreshold = useBufferRowSizeThreshold;
    this.useDirectRowUnpack = this.connection.getUseDirectRowUnpack();

    this.logSlowQueries = this.connection.getLogSlowQueries();

    this.reusablePacket = new Buffer(INITIAL_PACKET_SIZE);
    this.sendPacket = new Buffer(INITIAL_PACKET_SIZE);

    this.port = port;
    this.host = host;

    this.socketFactoryClassName = socketFactoryClassName;
    this.socketFactory = createSocketFactory();
    this.exceptionInterceptor = this.connection.getExceptionInterceptor();

    try {
        // 建立socket连接, 设置连接超时时间等配置，其中props是配置
        this.mysqlConnection = this.socketFactory.connect(this.host, this.port, props);

        if (socketTimeout != 0) {
            try {
                // 设置读超时时间
                this.mysqlConnection.setSoTimeout(socketTimeout);
            } catch (Exception ex) {
                /* Ignore if the platform does not support it */
            }
        }

        this.mysqlConnection = this.socketFactory.beforeHandshake();

        if (this.connection.getUseReadAheadInput()) {
            this.mysqlInput = new ReadAheadInputStream(this.mysqlConnection.getInputStream(), 16384, this.connection.getTraceProtocol(),
                    this.connection.getLog());
        } else if (this.connection.useUnbufferedInput()) {
            this.mysqlInput = this.mysqlConnection.getInputStream();
        } else {
            // 包装socket的输入流
            this.mysqlInput = new BufferedInputStream(this.mysqlConnection.getInputStream(), 16384);
        }
        // 包装socket的输出流
        this.mysqlOutput = new BufferedOutputStream(this.mysqlConnection.getOutputStream(), 16384);

        this.isInteractiveClient = this.connection.getInteractiveClient();
        this.profileSql = this.connection.getProfileSql();
        this.autoGenerateTestcaseScript = this.connection.getAutoGenerateTestcaseScript();

        this.needToGrabQueryFromPacket = (this.profileSql || this.logSlowQueries || this.autoGenerateTestcaseScript);

        if (this.connection.getUseNanosForElapsedTime() && TimeUtil.nanoTimeAvailable()) {
            this.useNanosForElapsedTime = true;

            this.queryTimingUnits = Messages.getString("Nanoseconds");
        } else {
            this.queryTimingUnits = Messages.getString("Milliseconds");
        }

        if (this.connection.getLogSlowQueries()) {
            calculateSlowQueryThreshold();
        }
    } catch (IOException ioEx) {
        throw SQLError.createCommunicationsException(this.connection, 0, 0, ioEx, getExceptionInterceptor());
    }
}
```

使用StandardSocketFactory来创建socket：mysqlConnection

```java
public Socket connect(String hostname, int portNumber, Properties props) throws SocketException, IOException {

    if (props != null) {
        this.host = hostname;

        this.port = portNumber;

        String localSocketHostname = props.getProperty("localSocketAddress");
        InetSocketAddress localSockAddr = null;
        if (localSocketHostname != null && localSocketHostname.length() > 0) {
            localSockAddr = new InetSocketAddress(InetAddress.getByName(localSocketHostname), 0);
        }

        String connectTimeoutStr = props.getProperty("connectTimeout");

        int connectTimeout = 0;

        if (connectTimeoutStr != null) {
            try {
                connectTimeout = Integer.parseInt(connectTimeoutStr);
            } catch (NumberFormatException nfe) {
                throw new SocketException("Illegal value '" + connectTimeoutStr + "' for connectTimeout");
            }
        }

        if (this.host != null) {
            InetAddress[] possibleAddresses = InetAddress.getAllByName(this.host);

            if (possibleAddresses.length == 0) {
                throw new SocketException("No addresses for host");
            }

            // save last exception to propagate to caller if connection fails
            SocketException lastException = null;

            // Need to loop through all possible addresses. Name lookup may return multiple addresses including IPv4 and IPv6 addresses. Some versions of
            // MySQL don't listen on the IPv6 address so we try all addresses.
            for (int i = 0; i < possibleAddresses.length; i++) {
                try {
                    // 创建socket
                    this.rawSocket = createSocket(props);

                    configureSocket(this.rawSocket, props);

                    InetSocketAddress sockAddr = new InetSocketAddress(possibleAddresses[i], this.port);
                    // bind to the local port if not using the ephemeral port
                    if (localSockAddr != null) {
                        this.rawSocket.bind(localSockAddr);
                    }

                    // 设置socket的连接超时时间
                    this.rawSocket.connect(sockAddr, getRealTimeout(connectTimeout));

                    break;
                } catch (SocketException ex) {
                    lastException = ex;
                    resetLoginTimeCountdown();
                    this.rawSocket = null;
                }
            }

            if (this.rawSocket == null && lastException != null) {
                throw lastException;
            }

            resetLoginTimeCountdown();

            return this.rawSocket;
        }
    }

    throw new SocketException("Unable to create socket");
}
```

创建socket

```java
protected Socket createSocket(Properties props) {
    return new Socket();
}
```



### 认证鉴权

利用com.mysql.jdbc.MysqlIO#doHandshake来进行鉴权

```java
void doHandshake(String user, String password, String database) throws SQLException {
    。。。

    //
    // switch to pluggable authentication if available
    //
    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH) != 0) {
        // 进行鉴权
        proceedHandshakeWithPluggableAuthentication(user, password, database, buf);
        return;
    }

    。。。
}
```

将密码使用SHA-1算法进行摘要加密运算，之后使用用户名、密码进行socket通信鉴权

```java
private void proceedHandshakeWithPluggableAuthentication(String user, String password, String database, Buffer challenge) throws SQLException {
        if (this.authenticationPlugins == null) {
            loadAuthenticationPlugins();
        }

        boolean skipPassword = false;
        int passwordLength = 16;
        int userLength = (user != null) ? user.length() : 0;
        int databaseLength = (database != null) ? database.length() : 0;

        int packLength = ((userLength + passwordLength + databaseLength) * 3) + 7 + HEADER_LENGTH + AUTH_411_OVERHEAD;

        AuthenticationPlugin plugin = null;
        Buffer fromServer = null;
        ArrayList<Buffer> toServer = new ArrayList<Buffer>();
        boolean done = false;
        Buffer last_sent = null;

        boolean old_raw_challenge = false;

        int counter = 100;

        while (0 < counter--) {

            if (!done) {

                if (challenge != null) {

                    if (challenge.isOKPacket()) {
                        throw SQLError.createSQLException(
                                Messages.getString("Connection.UnexpectedAuthenticationApproval", new Object[] { plugin.getProtocolPluginName() }),
                                getExceptionInterceptor());
                    }

                    // read Auth Challenge Packet

                    this.clientParam |= CLIENT_PLUGIN_AUTH | CLIENT_LONG_PASSWORD | CLIENT_PROTOCOL_41 | CLIENT_TRANSACTIONS // Need this to get server status values
                            | CLIENT_MULTI_RESULTS // We always allow multiple result sets
                            | CLIENT_SECURE_CONNECTION; // protocol with pluggable authentication always support this

                    // We allow the user to configure whether or not they want to support multiple queries (by default, this is disabled).
                    if (this.connection.getAllowMultiQueries()) {
                        this.clientParam |= CLIENT_MULTI_STATEMENTS;
                    }

                    if (((this.serverCapabilities & CLIENT_CAN_HANDLE_EXPIRED_PASSWORD) != 0) && !this.connection.getDisconnectOnExpiredPasswords()) {
                        this.clientParam |= CLIENT_CAN_HANDLE_EXPIRED_PASSWORD;
                    }
                    if (((this.serverCapabilities & CLIENT_CONNECT_ATTRS) != 0) && !NONE.equals(this.connection.getConnectionAttributes())) {
                        this.clientParam |= CLIENT_CONNECT_ATTRS;
                    }
                    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA) != 0) {
                        this.clientParam |= CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA;
                    }

                    this.has41NewNewProt = true;
                    this.use41Extensions = true;

                    if (this.connection.getUseSSL()) {
                        negotiateSSLConnection(user, password, database, packLength);
                    }

                    String pluginName = null;
                    // Due to Bug#59453 the auth-plugin-name is missing the terminating NUL-char in versions prior to 5.5.10 and 5.6.2.
                    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH) != 0) {
                        if (!versionMeetsMinimum(5, 5, 10) || versionMeetsMinimum(5, 6, 0) && !versionMeetsMinimum(5, 6, 2)) {
                            pluginName = challenge.readString("ASCII", getExceptionInterceptor(), this.authPluginDataLength);
                        } else {
                            pluginName = challenge.readString("ASCII", getExceptionInterceptor());
                        }
                    }

                    plugin = getAuthenticationPlugin(pluginName);
                    if (plugin == null) {
                        /*
                         * Use default if there is no plugin for pluginName.
                         */
                        plugin = getAuthenticationPlugin(this.clientDefaultAuthenticationPluginName);
                    } else if (pluginName.equals(Sha256PasswordPlugin.PLUGIN_NAME) && !isSSLEstablished() && this.connection.getServerRSAPublicKeyFile() == null
                            && !this.connection.getAllowPublicKeyRetrieval()) {
                        /*
                         * Fall back to default if plugin is 'sha256_password' but required conditions for this to work aren't met. If default is other than
                         * 'sha256_password' this will result in an immediate authentication switch request, allowing for other plugins to authenticate
                         * successfully. If default is 'sha256_password' then the authentication will fail as expected. In both cases user's password won't be
                         * sent to avoid subjecting it to lesser security levels.
                         */
                        plugin = getAuthenticationPlugin(this.clientDefaultAuthenticationPluginName);
                        skipPassword = !this.clientDefaultAuthenticationPluginName.equals(pluginName);
                    }

                    this.serverDefaultAuthenticationPluginName = plugin.getProtocolPluginName();

                    checkConfidentiality(plugin);
                    fromServer = new Buffer(StringUtils.getBytes(this.seed));
                } else {
                    // no challenge so this is a changeUser call
                    plugin = getAuthenticationPlugin(this.serverDefaultAuthenticationPluginName == null ? this.clientDefaultAuthenticationPluginName
                            : this.serverDefaultAuthenticationPluginName);

                    checkConfidentiality(plugin);

                    // Servers not affected by Bug#70865 expect the Change User Request containing a correct answer
                    // to seed sent by the server during the initial handshake, thus we reuse it here.
                    // Servers affected by Bug#70865 will just ignore it and send the Auth Switch.
                    fromServer = new Buffer(StringUtils.getBytes(this.seed));
                }

            } else {

                // 校验鉴权的socket发送过去之后，服务端返回值是否正确
                challenge = checkErrorPacket();
                old_raw_challenge = false;
                this.packetSequence++;
                this.compressedPacketSequence++;

                if (plugin == null) {
                    // this shouldn't happen in normal handshake packets exchange,
                    // we do it just to ensure that we don't get NPE in other case
                    plugin = getAuthenticationPlugin(this.serverDefaultAuthenticationPluginName != null ? this.serverDefaultAuthenticationPluginName
                            : this.clientDefaultAuthenticationPluginName);
                }

                if (challenge.isOKPacket()) {
                    // get the server status from the challenge packet.
                    challenge.newReadLength(); // affected_rows
                    challenge.newReadLength(); // last_insert_id

                    this.oldServerStatus = this.serverStatus;
                    this.serverStatus = challenge.readInt();

                    // if OK packet then finish handshake
                    plugin.destroy();
                    break;

                } else if (challenge.isAuthMethodSwitchRequestPacket()) {
                    skipPassword = false;

                    // read Auth Method Switch Request Packet
                    String pluginName = challenge.readString("ASCII", getExceptionInterceptor());

                    // get new plugin
                    if (!plugin.getProtocolPluginName().equals(pluginName)) {
                        plugin.destroy();
                        plugin = getAuthenticationPlugin(pluginName);
                        // if plugin is not found for pluginName throw exception
                        if (plugin == null) {
                            throw SQLError.createSQLException(Messages.getString("Connection.BadAuthenticationPlugin", new Object[] { pluginName }),
                                    getExceptionInterceptor());
                        }
                    }

                    checkConfidentiality(plugin);
                    fromServer = new Buffer(StringUtils.getBytes(challenge.readString("ASCII", getExceptionInterceptor())));

                } else {
                    // read raw packet
                    if (versionMeetsMinimum(5, 5, 16)) {
                        fromServer = new Buffer(challenge.getBytes(challenge.getPosition(), challenge.getBufLength() - challenge.getPosition()));
                    } else {
                        old_raw_challenge = true;
                        fromServer = new Buffer(challenge.getBytes(challenge.getPosition() - 1, challenge.getBufLength() - challenge.getPosition() + 1));
                    }
                }

            }

            // call plugin
            try {
                plugin.setAuthenticationParameters(user, skipPassword ? null : password);
                done = plugin.nextAuthenticationStep(fromServer, toServer);
            } catch (SQLException e) {
                throw SQLError.createSQLException(e.getMessage(), e.getSQLState(), e, getExceptionInterceptor());
            }

            // send response
            if (toServer.size() > 0) {
                if (challenge == null) {
                    String enc = getEncodingForHandshake();

                    // write COM_CHANGE_USER Packet
                    last_sent = new Buffer(packLength + 1);
                    last_sent.writeByte((byte) MysqlDefs.COM_CHANGE_USER);

                    // User/Password data
                    last_sent.writeString(user, enc, this.connection);

                    // 'auth-response-len' is limited to one Byte but, in case of success, COM_CHANGE_USER will be followed by an AuthSwitchRequest anyway
                    if (toServer.get(0).getBufLength() < 256) {
                        // non-mysql servers may use this information to authenticate without requiring another round-trip
                        last_sent.writeByte((byte) toServer.get(0).getBufLength());
                        last_sent.writeBytesNoNull(toServer.get(0).getByteBuffer(), 0, toServer.get(0).getBufLength());
                    } else {
                        last_sent.writeByte((byte) 0);
                    }

                    if (this.useConnectWithDb) {
                        last_sent.writeString(database, enc, this.connection);
                    } else {
                        /* For empty database */
                        last_sent.writeByte((byte) 0);
                    }

                    appendCharsetByteForHandshake(last_sent, enc);
                    // two (little-endian) bytes for charset in this packet
                    last_sent.writeByte((byte) 0);

                    // plugin name
                    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH) != 0) {
                        last_sent.writeString(plugin.getProtocolPluginName(), enc, this.connection);
                    }

                    // connection attributes
                    if ((this.clientParam & CLIENT_CONNECT_ATTRS) != 0) {
                        sendConnectionAttributes(last_sent, enc, this.connection);
                        last_sent.writeByte((byte) 0);
                    }

                    send(last_sent, last_sent.getPosition());

                } else if (challenge.isAuthMethodSwitchRequestPacket()) {
                    // write Auth Method Switch Response Packet
                    last_sent = new Buffer(toServer.get(0).getBufLength() + HEADER_LENGTH);
                    last_sent.writeBytesNoNull(toServer.get(0).getByteBuffer(), 0, toServer.get(0).getBufLength());
                    send(last_sent, last_sent.getPosition());

                } else if (challenge.isRawPacket() || old_raw_challenge) {
                    // write raw packet(s)
                    for (Buffer buffer : toServer) {
                        last_sent = new Buffer(buffer.getBufLength() + HEADER_LENGTH);
                        last_sent.writeBytesNoNull(buffer.getByteBuffer(), 0, toServer.get(0).getBufLength());
                        send(last_sent, last_sent.getPosition());
                    }

                } else {
                    // write Auth Response Packet
                    String enc = getEncodingForHandshake();

                    last_sent = new Buffer(packLength);
                    last_sent.writeLong(this.clientParam);
                    last_sent.writeLong(this.maxThreeBytes);

                    appendCharsetByteForHandshake(last_sent, enc);

                    last_sent.writeBytesNoNull(new byte[23]);	// Set of bytes reserved for future use.

                    // User/Password data
                    last_sent.writeString(user, enc, this.connection);

                    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA) != 0) {
                        // send lenenc-int length of auth-response and string[n] auth-response
                        last_sent.writeLenBytes(toServer.get(0).getBytes(toServer.get(0).getBufLength()));
                    } else {
                        // send 1 byte length of auth-response and string[n] auth-response
                        last_sent.writeByte((byte) toServer.get(0).getBufLength());
                        last_sent.writeBytesNoNull(toServer.get(0).getByteBuffer(), 0, toServer.get(0).getBufLength());
                    }

                    if (this.useConnectWithDb) {
                        last_sent.writeString(database, enc, this.connection);
                    } else {
                        /* For empty database */
                        last_sent.writeByte((byte) 0);
                    }

                    if ((this.serverCapabilities & CLIENT_PLUGIN_AUTH) != 0) {
                        last_sent.writeString(plugin.getProtocolPluginName(), enc, this.connection);
                    }

                    // connection attributes
                    if (((this.clientParam & CLIENT_CONNECT_ATTRS) != 0)) {
                        sendConnectionAttributes(last_sent, enc, this.connection);
                    }

                    send(last_sent, last_sent.getPosition());
                }

            }

        }

        if (counter == 0) {
            throw SQLError.createSQLException(Messages.getString("CommunicationsException.TooManyAuthenticationPluginNegotiations"), getExceptionInterceptor());
        }

        //
        // Can't enable compression until after handshake
        //
        if (((this.serverCapabilities & CLIENT_COMPRESS) != 0) && this.connection.getUseCompression() && !(this.mysqlInput instanceof CompressedInputStream)) {
            // The following matches with ZLIB's compress()
            this.deflater = new Deflater();
            this.useCompression = true;
            this.mysqlInput = new CompressedInputStream(this.connection, this.mysqlInput);
        }

        if (!this.useConnectWithDb) {
            changeDatabaseTo(database);
        }

        try {
            this.mysqlConnection = this.socketFactory.afterHandshake();
        } catch (IOException ioEx) {
            throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, ioEx,
                    getExceptionInterceptor());
        }
    }
```

通过socket发送数据到网络中

```java
private final void send(Buffer packet, int packetLen) throws SQLException {
    try {
        if (this.maxAllowedPacket > 0 && packetLen > this.maxAllowedPacket) {
            throw new PacketTooBigException(packetLen, this.maxAllowedPacket);
        }

        if ((this.serverMajorVersion >= 4) && (packetLen - HEADER_LENGTH >= this.maxThreeBytes
                || (this.useCompression && packetLen - HEADER_LENGTH >= this.maxThreeBytes - COMP_HEADER_LENGTH))) {
            sendSplitPackets(packet, packetLen);

        } else {
            this.packetSequence++;

            Buffer packetToSend = packet;
            packetToSend.setPosition(0);
            packetToSend.writeLongInt(packetLen - HEADER_LENGTH);
            packetToSend.writeByte(this.packetSequence);

            if (this.useCompression) {
                this.compressedPacketSequence++;
                int originalPacketLen = packetLen;

                packetToSend = compressPacket(packetToSend, 0, packetLen);
                packetLen = packetToSend.getPosition();

                if (this.traceProtocol) {
                    StringBuilder traceMessageBuf = new StringBuilder();

                    traceMessageBuf.append(Messages.getString("MysqlIO.57"));
                    traceMessageBuf.append(getPacketDumpToLog(packetToSend, packetLen));
                    traceMessageBuf.append(Messages.getString("MysqlIO.58"));
                    traceMessageBuf.append(getPacketDumpToLog(packet, originalPacketLen));

                    this.connection.getLog().logTrace(traceMessageBuf.toString());
                }
            } else {

                if (this.traceProtocol) {
                    StringBuilder traceMessageBuf = new StringBuilder();

                    traceMessageBuf.append(Messages.getString("MysqlIO.59"));
                    traceMessageBuf.append("host: '");
                    traceMessageBuf.append(this.host);
                    traceMessageBuf.append("' threadId: '");
                    traceMessageBuf.append(this.threadId);
                    traceMessageBuf.append("'\n");
                    traceMessageBuf.append(packetToSend.dump(packetLen));

                    this.connection.getLog().logTrace(traceMessageBuf.toString());
                }
            }

            // 将鉴权信息使用mysql通信协议的格式写到网络中
            this.mysqlOutput.write(packetToSend.getByteBuffer(), 0, packetLen);
            this.mysqlOutput.flush();
        }

        if (this.enablePacketDebug) {
            enqueuePacketForDebugging(true, false, packetLen + 5, this.packetHeaderBuf, packet);
        }

        //
        // Don't hold on to large packets
        //
        if (packet == this.sharedSendPacket) {
            reclaimLargeSharedSendPacket();
        }

        if (this.connection.getMaintainTimeStats()) {
            this.lastPacketSentTimeMs = System.currentTimeMillis();
        }
    } catch (IOException ioEx) {
        throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, ioEx,
                getExceptionInterceptor());
    }
}
```



在checkErrorPacket();检验服务端返回的信息是否正确

```java
protected Buffer checkErrorPacket() throws SQLException {
    return checkErrorPacket(-1);
}
```

```java
private Buffer checkErrorPacket(int command) throws SQLException {
    //int statusCode = 0;
    Buffer resultPacket = null;
    this.serverStatus = 0;

    try {
        // Check return value, if we get a java.io.EOFException, the server has gone away. We'll pass it on up the exception chain and let someone higher up
        // decide what to do (barf, reconnect, etc).
        // 校验鉴权结果
        resultPacket = reuseAndReadPacket(this.reusablePacket);
    } catch (SQLException sqlEx) {
        // Don't wrap SQL Exceptions
        throw sqlEx;
    } catch (Exception fallThru) {
        throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, fallThru,
                getExceptionInterceptor());
    }

    // 校验服务端返回的信息
    checkErrorPacket(resultPacket);

    return resultPacket;
}
```



```java
private final Buffer reuseAndReadPacket(Buffer reuse) throws SQLException {
    return reuseAndReadPacket(reuse, -1);
}
```

从socket中读数据，判断鉴权是否成功

```java
private final Buffer reuseAndReadPacket(Buffer reuse, int existingPacketLength) throws SQLException {

    try {
        reuse.setWasMultiPacket(false);
        int packetLength = 0;

        if (existingPacketLength == -1) {
            // 从socket中读数据
            int lengthRead = readFully(this.mysqlInput, this.packetHeaderBuf, 0, 4);

            if (lengthRead < 4) {
                forceClose();
                throw new IOException(Messages.getString("MysqlIO.43"));
            }

            packetLength = (this.packetHeaderBuf[0] & 0xff) + ((this.packetHeaderBuf[1] & 0xff) << 8) + ((this.packetHeaderBuf[2] & 0xff) << 16);
        } else {
            packetLength = existingPacketLength;
        }

        if (this.traceProtocol) {
            StringBuilder traceMessageBuf = new StringBuilder();

            traceMessageBuf.append(Messages.getString("MysqlIO.44"));
            traceMessageBuf.append(packetLength);
            traceMessageBuf.append(Messages.getString("MysqlIO.45"));
            traceMessageBuf.append(StringUtils.dumpAsHex(this.packetHeaderBuf, 4));

            this.connection.getLog().logTrace(traceMessageBuf.toString());
        }

        byte multiPacketSeq = this.packetHeaderBuf[3];

        if (!this.packetSequenceReset) {
            if (this.enablePacketDebug && this.checkPacketSequence) {
                checkPacketSequencing(multiPacketSeq);
            }
        } else {
            this.packetSequenceReset = false;
        }

        this.readPacketSequence = multiPacketSeq;

        // Set the Buffer to it's original state
        reuse.setPosition(0);

        // Do we need to re-alloc the byte buffer?
        //
        // Note: We actually check the length of the buffer, rather than getBufLength(), because getBufLength() is not necesarily the actual length of the
        // byte array used as the buffer
        if (reuse.getByteBuffer().length <= packetLength) {
            reuse.setByteBuffer(new byte[packetLength + 1]);
        }

        // Set the new length
        reuse.setBufLength(packetLength);

        // Read the data from the server
        int numBytesRead = readFully(this.mysqlInput, reuse.getByteBuffer(), 0, packetLength);

        if (numBytesRead != packetLength) {
            throw new IOException("Short read, expected " + packetLength + " bytes, only read " + numBytesRead);
        }

        if (this.traceProtocol) {
            StringBuilder traceMessageBuf = new StringBuilder();

            traceMessageBuf.append(Messages.getString("MysqlIO.46"));
            traceMessageBuf.append(getPacketDumpToLog(reuse, packetLength));

            this.connection.getLog().logTrace(traceMessageBuf.toString());
        }

        if (this.enablePacketDebug) {
            enqueuePacketForDebugging(false, true, 0, this.packetHeaderBuf, reuse);
        }

        boolean isMultiPacket = false;

        if (packetLength == this.maxThreeBytes) {
            reuse.setPosition(this.maxThreeBytes);

            // it's multi-packet
            isMultiPacket = true;

            packetLength = readRemainingMultiPackets(reuse, multiPacketSeq);
        }

        if (!isMultiPacket) {
            reuse.getByteBuffer()[packetLength] = 0; // Null-termination
        }

        if (this.connection.getMaintainTimeStats()) {
            this.lastPacketReceivedTimeMs = System.currentTimeMillis();
        }

        return reuse;
    } catch (IOException ioEx) {
        throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, ioEx,
                getExceptionInterceptor());
    } catch (OutOfMemoryError oom) {
        try {
            // _Try_ this
            clearInputStream();
        } catch (Exception ex) {
        }
        try {
            this.connection.realClose(false, false, true, oom);
        } catch (Exception ex) {
        }
        throw oom;
    }

}
```



从socket中读取数据

```java
private final int readFully(InputStream in, byte[] b, int off, int len) throws IOException {
    if (len < 0) {
        throw new IndexOutOfBoundsException();
    }

    int n = 0;

    while (n < len) {
        int count = in.read(b, off + n, len - n);

        if (count < 0) {
            throw new EOFException(Messages.getString("MysqlIO.EOF", new Object[] { Integer.valueOf(len), Integer.valueOf(n) }));
        }

        n += count;
    }

    return n;
}
```

使用checkErrorPacket检验服务端返回的错误信息

```java
private void checkErrorPacket(Buffer resultPacket) throws SQLException {

    int statusCode = resultPacket.readByte();

    // 处理错误
    if (statusCode == (byte) 0xff) {
        String serverErrorMessage;
        int errno = 2000;

        if (this.protocolVersion > 9) {
            errno = resultPacket.readInt();

            String xOpen = null;

            // 拿到服务端返回的错误信息
            serverErrorMessage = resultPacket.readString(this.connection.getErrorMessageEncoding(), getExceptionInterceptor());

            if (serverErrorMessage.charAt(0) == '#') {

                // we have an SQLState
                if (serverErrorMessage.length() > 6) {
                    xOpen = serverErrorMessage.substring(1, 6);
                    serverErrorMessage = serverErrorMessage.substring(6);

                    if (xOpen.equals("HY000")) {
                        xOpen = SQLError.mysqlToSqlState(errno, this.connection.getUseSqlStateCodes());
                    }
                } else {
                    xOpen = SQLError.mysqlToSqlState(errno, this.connection.getUseSqlStateCodes());
                }
            } else {
                xOpen = SQLError.mysqlToSqlState(errno, this.connection.getUseSqlStateCodes());
            }

            clearInputStream();

            StringBuilder errorBuf = new StringBuilder();

            // 根据服务端返回的错误码，客户端从缓存中拿到错误信息
            String xOpenErrorMessage = SQLError.get(xOpen);

            // 只使用服务端返回的错误信息，默认是只使用服务单错误信息
            if (!this.connection.getUseOnlyServerErrorMessages()) {
                if (xOpenErrorMessage != null) {
                    errorBuf.append(xOpenErrorMessage);
                    errorBuf.append(Messages.getString("MysqlIO.68"));
                }
            }

            errorBuf.append(serverErrorMessage);

            if (!this.connection.getUseOnlyServerErrorMessages()) {
                if (xOpenErrorMessage != null) {
                    errorBuf.append("\"");
                }
            }

            appendDeadlockStatusInformation(xOpen, errorBuf);

            if (xOpen != null && xOpen.startsWith("22")) {
                throw new MysqlDataTruncation(errorBuf.toString(), 0, true, false, 0, 0, errno);
            }
            throw SQLError.createSQLException(errorBuf.toString(), xOpen, errno, false, getExceptionInterceptor(), this.connection);
        }

        serverErrorMessage = resultPacket.readString(this.connection.getErrorMessageEncoding(), getExceptionInterceptor());
        clearInputStream();

        if (serverErrorMessage.indexOf(Messages.getString("MysqlIO.70")) != -1) {
            throw SQLError.createSQLException(SQLError.get(SQLError.SQL_STATE_COLUMN_NOT_FOUND) + ", " + serverErrorMessage,
                    SQLError.SQL_STATE_COLUMN_NOT_FOUND, -1, false, getExceptionInterceptor(), this.connection);
        }

        StringBuilder errorBuf = new StringBuilder(Messages.getString("MysqlIO.72"));
        errorBuf.append(serverErrorMessage);
        errorBuf.append("\"");

        throw SQLError.createSQLException(SQLError.get(SQLError.SQL_STATE_GENERAL_ERROR) + ", " + errorBuf.toString(), SQLError.SQL_STATE_GENERAL_ERROR, -1,
                false, getExceptionInterceptor(), this.connection);
    }
}
```



## 发送数据

```java
final Buffer sendCommand(int command, String extraData, Buffer queryPacket, boolean skipCheck, String extraDataCharEncoding, int timeoutMillis)
        throws SQLException {
    this.commandCount++;

    //
    // We cache these locally, per-command, as the checks for them are in very 'hot' sections of the I/O code and we save 10-15% in overall performance by
    // doing this...
    //
    this.enablePacketDebug = this.connection.getEnablePacketDebug();
    this.readPacketSequence = 0;

    int oldTimeout = 0;

    if (timeoutMillis != 0) {
        try {
            oldTimeout = this.mysqlConnection.getSoTimeout();
            this.mysqlConnection.setSoTimeout(timeoutMillis);
        } catch (SocketException e) {
            throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, e,
                    getExceptionInterceptor());
        }
    }

    try {

        checkForOutstandingStreamingData();

        // Clear serverStatus...this value is guarded by an external mutex, as you can only ever be processing one command at a time
        this.oldServerStatus = this.serverStatus;
        this.serverStatus = 0;
        this.hadWarnings = false;
        this.warningCount = 0;

        this.queryNoIndexUsed = false;
        this.queryBadIndexUsed = false;
        this.serverQueryWasSlow = false;

        //
        // Compressed input stream needs cleared at beginning of each command execution...
        //
        if (this.useCompression) {
            int bytesLeft = this.mysqlInput.available();

            if (bytesLeft > 0) {
                this.mysqlInput.skip(bytesLeft);
            }
        }

        try {
            clearInputStream();

            //
            // PreparedStatements construct their own packets, for efficiency's sake.
            //
            // If this is a generic query, we need to re-use the sending packet.
            // 发送数据
            if (queryPacket == null) {
                int packLength = HEADER_LENGTH + COMP_HEADER_LENGTH + 1 + ((extraData != null) ? extraData.length() : 0) + 2;

                if (this.sendPacket == null) {
                    this.sendPacket = new Buffer(packLength);
                }

                this.packetSequence = -1;
                this.compressedPacketSequence = -1;
                this.readPacketSequence = 0;
                this.checkPacketSequence = true;
                this.sendPacket.clear();

                this.sendPacket.writeByte((byte) command);

                if ((command == MysqlDefs.INIT_DB) || (command == MysqlDefs.QUERY) || (command == MysqlDefs.COM_PREPARE)) {
                    if (extraDataCharEncoding == null) {
                        this.sendPacket.writeStringNoNull(extraData);
                    } else {
                        this.sendPacket.writeStringNoNull(extraData, extraDataCharEncoding, this.connection.getServerCharset(),
                                this.connection.parserKnowsUnicode(), this.connection);
                    }
                }

                 // 发送数据
                send(this.sendPacket, this.sendPacket.getPosition());
            } else {
                this.packetSequence = -1;
                this.compressedPacketSequence = -1;
                 // 发送数据
                send(queryPacket, queryPacket.getPosition()); // packet passed by PreparedStatement
            }
        } catch (SQLException sqlEx) {
            // don't wrap SQLExceptions
            throw sqlEx;
        } catch (Exception ex) {
            throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, ex,
                    getExceptionInterceptor());
        }

        Buffer returnPacket = null;

        if (!skipCheck) {
            if ((command == MysqlDefs.COM_EXECUTE) || (command == MysqlDefs.COM_RESET_STMT)) {
                this.readPacketSequence = 0;
                this.packetSequenceReset = true;
            }
             // 校验返回值
            returnPacket = checkErrorPacket(command);
        }

        return returnPacket;
    } catch (IOException ioEx) {
        preserveOldTransactionState();
        throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, ioEx,
                getExceptionInterceptor());

    } catch (SQLException e) {
        preserveOldTransactionState();
        throw e;

    } finally {
        if (timeoutMillis != 0) {
            try {
                this.mysqlConnection.setSoTimeout(oldTimeout);
            } catch (SocketException e) {
                throw SQLError.createCommunicationsException(this.connection, this.lastPacketSentTimeMs, this.lastPacketReceivedTimeMs, e,
                        getExceptionInterceptor());
            }
        }
    }
}
```

发送数据都会调用sendCommand方法，在这个方法里面进行阻塞通信。

## 总结

1. 首先使用BIO创建socket连接，建立客户端与服务端的连接（设置的connectTimeout创建连接的超时时间）
2. 进行鉴权
   1. 客户端使用mysql协议发送数据到服务端（密码使用sha-1摘要算法加密）
   2. 服务端发送数据给客户端
   3. 客户端使用BIO阻塞获取服务端的数据（设置的socketTimeout读超时时间）并检验返回结果，确保鉴权成功
3. sql执行的时候也是使用NIO，客户端发数据，服务端返回数据，客户端检验数据正确后，处理数据

## 参考

https://blog.csdn.net/c929833623lvcha/article/details/44517245