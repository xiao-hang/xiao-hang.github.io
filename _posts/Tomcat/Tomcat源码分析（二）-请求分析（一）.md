# Tomcat 源码分析-请求分析（二）

[TOC]

------

## 一.处理线程的产生

#### 了解一下大体的线程情况

在默认配置下的Tomcat启动之后，后有一下的线程。其中1个用户线程，剩下5个为守护线程（如下图所示）。 

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614b60773dd2a06?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1) 

接下来，要说明的就是这个`http-bio-8080 ` 开头的两个守护线程（即 http-bio-8080-Acceptor-0 和 http-bio-8080-AsyncTimeout ） 

### 初始化各个必要的对象

####  加载配置信息，创建Connector节点

**默认的这个线程的配置在`server.xml `文件中 ：【对应节点 Server/Service/Connector 节点 】**

```xml
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

**所以，首先Tomcat 容器启动的时候 Digester  会读取这个配置文件，**产生相应的组件对象并采取链式调用的方式调用它们的 init 和 start 方法 。

**connector 节点时是这么处理配置是这样的：**

```java
digester.addRule("Server/Service/Connector",
                 new ConnectorCreateRule());
digester.addRule("Server/Service/Connector",
                 new SetAllPropertiesRule(new String[]{"executor"}));
digester.addSetNext("Server/Service/Connector",
                    "addConnector",
                    "org.apache.catalina.connector.Connector");
```

这里创建节点的时候会调用规则类ConnectorCreateRule，这里ConnectorCreateRule.begin()方法会被调用，用以创建Connector 节点 。

ConnectorCreateRule.begin()方法如下：

```java
  @Override
    public void begin(String namespace, String name, Attributes attributes)
            throws Exception {
        Service svc = (Service)digester.peek();
        Executor ex = null;
        if ( attributes.getValue("executor")!=null ) {
            ex = svc.getExecutor(attributes.getValue("executor"));
        }
        //这里节点protocol的配置作为参数（HTTP/1.1）去创建Connector节点对象
        Connector con = new Connector(attributes.getValue("protocol")); 
        if ( ex != null )  _setExecutor(con,ex);
        digester.push(con);
    }
```

#### Connector节点构造方法，创建Http11Protocol

**构造方法传入参数** ：`HTTP/1.1`

```java
public Connector(String protocol) {
    setProtocol(protocol);
    // Instantiate protocol handler
    try {
        Class<?> clazz = Class.forName(protocolHandlerClassName);
        this.protocolHandler = (ProtocolHandler) clazz.getDeclaredConstructor().newInstance();
    } catch (Exception e) {
        log.error(sm.getString(
                "coyoteConnector.protocolHandlerInstantiationFailed"), e);
    }
}
```

其中，这里会**先执行Connector .setProtocol方法**，参数为`HTTP/1.1`。方法代码如下：

```java
    public void setProtocol(String protocol) {
        if (AprLifecycleListener.isAprAvailable()) {  //这个值默认为false的样子
            if ("HTTP/1.1".equals(protocol)) {
                setProtocolHandlerClassName
                ("org.apache.coyote.http11.Http11AprProtocol");
            } else if ("AJP/1.3".equals(protocol)) {
                setProtocolHandlerClassName
                ("org.apache.coyote.ajp.AjpAprProtocol");
            } else if (protocol != null) {
                setProtocolHandlerClassName(protocol);
            } else {
                setProtocolHandlerClassName
                ("org.apache.coyote.http11.Http11AprProtocol");
            }
        } else {
            if ("HTTP/1.1".equals(protocol)) {
                setProtocolHandlerClassName
                ("org.apache.coyote.http11.Http11Protocol");  //按照参数最后会到这里
            } else if ("AJP/1.3".equals(protocol)) {
                setProtocolHandlerClassName
                ("org.apache.coyote.ajp.AjpProtocol");
            } else if (protocol != null) {
                setProtocolHandlerClassName(protocol);
            }
        }
    }
```

然后，Connector 类实例变量 protocolHandlerClassName 值设置为`org.apache.coyote.http11.Http11Protocol` ，构造方法接下来的就是使用**反射来产生Http11Protocol 的对象**：

```java
 		Class<?> clazz = Class.forName(protocolHandlerClassName);
        this.protocolHandler = (ProtocolHandler) clazz.getDeclaredConstructor().newInstance();
```

#### Http11Protocol 对象的产生

**Http11Protocol 的构造方法：产生了一个**JIoEndpoint对象

```java
    public Http11Protocol() {
        endpoint = new JIoEndpoint();
        cHandler = new Http11ConnectionHandler(this);
        ((JIoEndpoint) endpoint).setHandler(cHandler);
        setSoLinger(Constants.DEFAULT_CONNECTION_LINGER);
        setSoTimeout(Constants.DEFAULT_CONNECTION_TIMEOUT);
        setTcpNoDelay(Constants.DEFAULT_TCP_NO_DELAY);
    }
```

这里，产生了一个JIoEndpoint对象。

到这里，所有的构造方法调用结束，该产生的对象也已经创建好了。

大概图解一下构造的调用的过程：

```sequence
Title: 构造的调用的过程
Note left of Connector: server.xml
Note left of Connector: Digester 加载 
Note over Connector: ConnectorCreateRule 创建
Connector->Http11Protocol: 构造方法，实际初始化实
Http11Protocol->JIoEndpoint: 构造方法，实际初始化实
JIoEndpoint-->Http11Protocol: 作为类变量(在父类中定义)
Note over Http11Protocol: endpoint变量
```

### 执行start方法

> 按照Tomcat 的启动方式，在创建完节点对象时候，会链式的调用start方法到具体实现类的startInternal()方法

####  Connector 类的 startInternal  方法

先看源码：

```java
protected void startInternal() throws LifecycleException {
    // Validate settings before starting
    if (getPort() < 0) {
        throw new LifecycleException(sm.getString(
                "coyoteConnector.invalidPort", Integer.valueOf(getPort())));
    }
    setState(LifecycleState.STARTING);
    try {
        protocolHandler.start();   //对，就看这里这个就行了 (个_个)
    } catch (Exception e) {
        String errPrefix = "";
        if(this.service != null) {
            errPrefix += "service.getName(): \"" + this.service.getName() + "\"; ";
        }
        throw new LifecycleException
        (errPrefix + " " + sm.getString
                ("coyoteConnector.protocolHandlerStartFailed"), e);
    }
    mapperListener.start();
}
```

 这里可以知道，调用的是` protocolHandler.start(); `,然而，这个protocolHandler在其构造方法中就被赋值Http11Protocol 对象了。

#### 所以，这里调用的是Http11Protocol  类的 star() 方法

> 这里start方法在父类org.apache.coyote.AbstractProtocol 中

```java
public void start() throws Exception {
    if (getLog().isInfoEnabled())
        getLog().info(sm.getString("abstractProtocolHandler.start",
                getName()));
    try {
        endpoint.start();//这里，这里，又调用了一个start方法
    } catch (Exception ex) {
        getLog().error(sm.getString("abstractProtocolHandler.startError",
                getName()), ex);
        throw ex;
    }
}
```

这里会调用endpoint的start()方法。

endpoint  是`org.apache.tomcat.util.net.JIoEndpoint`类的实例 。

#### 最终会执行该类的 JIoEndpoint.startInternal 方法

> 这里，就是产生开头说明的两个线程的地方了，(ノ｀Д)ノ调用了这么久终于到地方了。

```java
    @Override
    public void startInternal() throws Exception {
        if (!running) {
            running = true;
            paused = false;
            // Create worker collection
            if (getExecutor() == null) {
                createExecutor();
            }
            initializeConnectionLatch();
            startAcceptorThreads(); //这个启动http-bio-8080-Acceptor-0 线程
            // Start async timeout thread
            //启动 http-bio-8080-AsyncTimeout 线程
            Thread timeoutThread = new Thread(new AsyncTimeout(),
                    getName() + "-AsyncTimeout");
            timeoutThread.setPriority(threadPriority);
            timeoutThread.setDaemon(true);
            timeoutThread.start();
        }
    }
```

startAcceptorThreads（）方法如下：启动http-bio-8080-Acceptor-0 线程

> 这个方法在AbstractEndpoint中，而createAcceptor() 方法则在实现类JIoEndpoint中；

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
    /**
     * Hook to allow Endpoints to provide a specific Acceptor implementation.
     */
    protected abstract Acceptor createAcceptor();
```

### 结束，开启线程了

到这里，线程已经启动了，然而，具体线程的操作呢，这个在最后说明的，JIoEndpoint类的实现方法createAcceptor()中。这个接下来再具体学习一下。

（对了，ajp-bio-8009-Acceptor-0 和 ajp-bio-8009-AsyncTimeout 两个守护线程的产生和启动方式也差不多这样子╮(╯_╰)╭ ）

#### 先来试试画一个，启动线程各个类的调用图，丫的(ノ｀Д)ノ真的是很乱

```sequence
title: Start启动调用流程图
note left of Connector: Start
note over Connector:startInternal()初始化方法
Connector->Http11Protocol: protocolHandler.start();
Http11Protocol-->AbstractProtocol: 调用父类方法
note over AbstractProtocol:Start()方法 
AbstractProtocol->JIoEndpoint:endpoint.start();
note over JIoEndpoint:startInternal()
note over JIoEndpoint: startAcceptorThreads() 
JIoEndpoint->AbstractEndpoint:调用父类方法
note over AbstractEndpoint: createAcceptor()方法定义
AbstractEndpoint-->JIoEndpoint:实现方法
note over JIoEndpoint: Acceptor
note over JIoEndpoint: 【这里是具体的线程执行的内容】
JIoEndpoint-->AbstractEndpoint:返回线程操作
note over AbstractEndpoint: 线程启动 ;
note over JIoEndpoint: 线程启动 timeoutThread.start();
```

2019-01-08 

---

---

## 二.Socket 转换为内部请求对象-request

#### 一些的瞎说明

Tomcat 作为Java实现的一种WEB服务，WEB服务的解释是：在网络环境下可以向发出请求的浏览器提供文档的程序 ；

辣么，这里使用Java实现，进行网络通信必然是用到Socket编程，服务作为Socket服务端，浏览器作为Socket客户端。浏览器与服务器的一次交互就分四步：连接，请求，响应，关闭。接下来就具体看下Tomcat是如何实现的。

### Socket 请求连接监听

> 这里承接上一篇文章，已经分析道，Connector 节点创建启动的最后，作为线程启动的具体实现类是`org.apache.tomcat.util.net.JIoEndpoint.Acceptor ` 

JIoEndpoint.Acceptor 类的工作：作为监听传入TCP/IP连接的后台线程并将它们交给适当的处理器；

源码如下：（会删减掉一些异常处理等一堆细节代码）

```java
    protected class Acceptor extends AbstractEndpoint.Acceptor {
        @Override
        public void run() {
            int errorDelay = 0;
            // Loop until we receive a shutdown command
            while (running) {
                // Loop if endpoint is paused
                while (paused && running) {
                    state = AcceptorState.PAUSED;
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }
                if (!running) {
                    break;
                }
                state = AcceptorState.RUNNING;
                try {
                    //if we have reached max connections, wait
                    countUpOrAwaitConnection();
                    Socket socket = null;
                    try {
                        //这里是第一个要注意的地方
                        // Accept the next incoming connection from the server
                        //接受服务器的下一个传入连接(具体怎么来的，我现在也不知道╮(╯_╰)╭)
                        socket = serverSocketFactory.acceptSocket(serverSocket);
                    } catch (IOException ioe) {
                        countDownConnection();
                        // Introduce delay if necessary
                        errorDelay = handleExceptionWithDelay(errorDelay);
                        // re-throw
                        throw ioe;
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // Configure the socket
                    if (running && !paused && setSocketOptions(socket)) {
                        // Hand this socket off to an appropriate processor
                      //这里是第二个，会把连接的socket给处理器进行处理↓
                        //就是processSocket（）方法 【也在JIoEndpoint类中】
                        if (!processSocket(socket)) {
                            countDownConnection();
                            // Close socket right away
                            closeSocket(socket);
                        }
                    } else {
                        countDownConnection();
                        // Close socket right away
                        closeSocket(socket);
                    }
                } catch ........
            }
            state = AcceptorState.ENDED;
        }
    }
```

以上源码中就两个步骤，获取新来的Socket连接，然后把它对给处理方`processSocket`；

**processSocket方法的源码像这样：**

```java
//处理来自客户机的新连接，并设置keep-alive等属性，并对给执行类进行处理
    protected boolean processSocket(Socket socket) {
        // Process the request from this socket
        try {
            //这里Socket包装成SocketWrapper，并设置一些东东。
            SocketWrapper<Socket> wrapper = new SocketWrapper<Socket>(socket);
            wrapper.setKeepAliveLeft(getMaxKeepAliveRequests());
            wrapper.setSecure(isSSLEnabled());
            // During shutdown, executor may be null - avoid NPE
            if (!running) {
                return false;
            }
            //这里对给SocketProcessor类去处理了，
            //这里是另外启动一个线程来处理，以保证并发情况下的另一个请求的响应
            getExecutor().execute(new SocketProcessor(wrapper));
        } catch 。。。。。。。。
        return true;
    }
```

方法所做的事情就是把Socket打包为SocketWrapper，然后新开一个线程去处理这个连接。处理类为SocketProcessor；

### 启动新线程处理Socket的方法调用

**处理类`SocketProcessor`作为处理类，是作为新线程执行的。**

代码如下：

```java
    protected class SocketProcessor implements Runnable {
        protected SocketWrapper<Socket> socket = null;
        protected SocketStatus status = null;
        public SocketProcessor(SocketWrapper<Socket> socket) {
            if (socket==null) throw new NullPointerException();
            this.socket = socket;
        }

        @Override
        public void run() {
            boolean launch = false;
            synchronized (socket) {
                try {
                    SocketState state = SocketState.OPEN;
                    try {
                        // SSL handshake
                        serverSocketFactory.handshake(socket.getSocket());
                    } catch (Throwable t) {... }

                    if ((state != SocketState.CLOSED)) {
                        if (status == null) {
                            //通常是会跑到这里，给Http11ConnectionHandler。process处理了
                            state = handler.process(socket, SocketStatus.OPEN_READ);
                        } else {
                            state = handler.process(socket,status);
                        }
                    }
                    if (state == SocketState.CLOSED) {
                      。。。。。。。。。
                    } else if (state == SocketState.OPEN ||
                            state == SocketState.UPGRADING ||
                            state == SocketState.UPGRADING_TOMCAT  ||
                            state == SocketState.UPGRADED){
                        socket.setKeptAlive(true);
                        socket.access();
                        launch = true;
                    } else if (state == SocketState.LONG) {
                        socket.access();
                        waitingRequests.add(socket);
                    }
                } finally ......
            }
            socket = null;
            // Finish up this request
        }
    }
```

SocketProcessor作为启动新线程来处理连接，这里在判断结束之后，又丢给了handler.process来处理，这里的handler就是`Http11ConnectionHandler`，在`Http11Protocol `的构造方法中定义的：

```java
public Http11Protocol() {
    endpoint = new JIoEndpoint();
    cHandler = new Http11ConnectionHandler(this);
    ((JIoEndpoint) endpoint).setHandler(cHandler);
	。。。。。
}
```

而`.process`方法，则在`Http11ConnectionHandler`的父类`AbstractConnectionHandler`中定义的。

 **丢到AbstractConnectionHandler.process方法来处理数据流，方法的源码像这样：↓**

> 代码太长了，只保留点重点的代码，具体的可以去查看Tomcat源码

```java
        @SuppressWarnings("deprecation") // Old HTTP upgrade method has been deprecated
        public SocketState process(SocketWrapper<S> wrapper,
                SocketStatus status) {  //按照上面传进来的数据是OPEN_READ
            if (wrapper == null) {
                return SocketState.CLOSED;
            }
            S socket = wrapper.getSocket();
            if (socket == null) 
                return SocketState.CLOSED;
            }
            Processor<S> processor = connections.get(socket);
            if (status == SocketStatus.DISCONNECT && processor == null) {             
                return SocketState.CLOSED;
            }
            wrapper.setAsync(false);
            ContainerThreadMarker.markAsContainerThread();

            try {
                if (processor == null) {
                    processor = recycledProcessors.poll();
                }
                if (processor == null) {
                    //这里的时候会调用createProcessor来创建processor对象
                    //而这个方法的实现在子类中，这里是Http11ConnectionHandler中 
                    processor = createProcessor();
                }
                initSsl(wrapper, processor);
                SocketState state = SocketState.CLOSED;
                do {
                    if (status == SocketStatus.DISCONNECT &&
                            !processor.isComet()) {
                    } else if (processor.isAsync() || state == SocketState.ASYNC_END) {
                        state = processor.asyncDispatch(status);
                        if (state == SocketState.OPEN) {  //接下在会执行到这里
                            getProtocol().endpoint.removeWaitingRequest(wrapper);
                            //然后这里就是具体的处理连接数据的地方了
                            state = processor.process(wrapper);
                        }
                    } else if 
                        。。。。。
                } while (state == SocketState.ASYNC_END ||
                        state == SocketState.UPGRADING ||
                        state == SocketState.UPGRADING_TOMCAT);

                if (state == SocketState.LONG) {。。。
                } else if (state == SocketState.OPEN) {。。。
                } else if (state == SocketState.SENDFILE) {。。。
                } else if (state == SocketState.UPGRADED) {。。。
                } else {。。。。}
                return state;
            } catch。。。。。
            connections.remove(socket);
            // Don't try to add upgrade processors back into the pool
            if (!(processor instanceof org.apache.coyote.http11.upgrade.UpgradeProcessor)
                    && !processor.isUpgrade()) {
                release(wrapper, processor, true, false);
            }
            return SocketState.CLOSED;
        }
```

从上面↑的代码，可以看到，首先是调用createProcessor  来创建处理对象processor  ，而这个方法在这里只是个抽象方法，具体的实现在Http11ConnectionHandler  类中：

```java
 @Override
        protected Http11Processor createProcessor() {
            Http11Processor processor = new Http11Processor(
                    proto.getMaxHttpHeaderSize(), proto.getRejectIllegalHeaderName(),
                    (JIoEndpoint)proto.endpoint, proto.getMaxTrailerSize(),
                    proto.getAllowedTrailerHeadersAsSet(), proto.getMaxExtensionSize(),
                    proto.getMaxSwallowSize(), proto.getRelaxedPathChars(),
                    proto.getRelaxedQueryChars());
            processor.setAdapter(proto.adapter);
            processor.setMaxKeepAliveRequests(proto.getMaxKeepAliveRequests());
            processor.setKeepAliveTimeout(proto.getKeepAliveTimeout());
            processor.setConnectionUploadTimeout(
                    proto.getConnectionUploadTimeout());
            processor.setDisableUploadTimeout(proto.getDisableUploadTimeout());
            processor.setCompressionMinSize(proto.getCompressionMinSize());
            processor.setCompression(proto.getCompression());
            processor.setNoCompressionUserAgents(proto.getNoCompressionUserAgents());
            processor.setCompressableMimeTypes(proto.getCompressableMimeTypes());
            processor.setRestrictedUserAgents(proto.getRestrictedUserAgents());
            processor.setSocketBuffer(proto.getSocketBuffer());
            processor.setMaxSavePostSize(proto.getMaxSavePostSize());
            processor.setServer(proto.getServer());
            processor.setDisableKeepAlivePercentage(
                    proto.getDisableKeepAlivePercentage());
            processor.setMaxCookieCount(proto.getMaxCookieCount());
            register(processor);
            return processor;
        }
```

这个没什么解释的了，创建了Http11Processor类的实例，而依据前文代码中：会执行到的处理方法↓

```java
handler.process(socket, SocketStatus.OPEN_READ);
```

所以，这里调用的方法实际上是：Http11Processor.process方法；

> 当然，process方法的实现并不在Http11Processor中，而是在父类org.apache.coyote.http11.AbstractHttp11Processor 中实现的。

**处理方法的调用，到这里完成了。**【具体处理方法接下来分析】

**先来理清楚一下，上面提到的乱七八糟的调用：**

```sequence
title: 处理方法调用流程
note right of Acceptor(): JIoEndpoint类
note over Acceptor():获取socket
Acceptor()->processSocket(): if (!processSocket(socket)) {
note over processSocket(): 包装成SocketWrappe
note over processSocket(): 并设置一些东东
processSocket()->SocketProcessor类: 另外启动线程来处理连接
note over processSocket(), SocketProcessor类: getExecutor().execute(
note over processSocket(), SocketProcessor类: new SocketProcessor(wrapper));
note over SocketProcessor类: 作为新线程执行的处理类
note over SocketProcessor类: run()方法被执行
SocketProcessor类->Http11ConnectionHandler类: 调process()处理方法
note over SocketProcessor类,Http11ConnectionHandler类: handler.process(socket,OPEN_READ)
note over Http11ConnectionHandler类: process()
note over Http11ConnectionHandler类: 调用createProcessor()
Http11ConnectionHandler类->Http11Processor类: processor = createProcessor();
note left of Http11Processor类: 创建并返回Http11Processor类实例
Http11Processor类-->Http11ConnectionHandler类: 
note over Http11ConnectionHandler类: 最终执行processor.process(wrapper)
```

【20190312 继续学习】

### 具体的处理Socket数据流方法分析

> 具体的处理方法是`org.apache.coyote.http11.AbstractHttp11Processor `类的 `process `方法 
>
> 在这个方法里面处理的hhttp请求的字节流信息，并且封装到了内置对象request中。

具体的代码如下：【源码太多的处理，这里忽略很多，只保留用以分析的部分】

```java
 @Override
    public SocketState process(SocketWrapper<S> socketWrapper)
        throws IOException {
        RequestInfo rp = request.getRequestProcessor();
        rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

        // Setting up the I/O 从 Socket 中获取输入输出流
        //getInputBuffer这返回了inputBuffer ，在构造方法中已经初始化了 
        //构造方法:inputBuffer = new InternalInputBuffer(request, headerBufferSize);
    	//构造方法:request.setInputBuffer(inputBuffer);
        setSocketWrapper(socketWrapper);
        getInputBuffer().init(socketWrapper, endpoint);
        getOutputBuffer().init(socketWrapper, endpoint);

        // Flags
		//....

        while (!getErrorState().isError() && keepAlive && !comet && !isAsync() &&
                upgradeInbound == null &&
                httpUpgradeHandler == null && !endpoint.isPaused()) {

            // Parsing the request header 解析请求行和请求头 【这里就是重点了】
            try {
                setRequestLineReadTimeout();
					//这里 就是处理请求行的方法了↓↓↓↓↓↓↓↓↓↓
                if (!getInputBuffer().parseRequestLine(keptAlive)) {
                    if (handleIncompleteRequestLineRead()) {
                        break;
                    }
                }

                if (endpoint.isPaused()) {
                  。。。。。。
                } else {
              	 。。。。。。
                     //这里 就是处理请求头的方法了↓↓↓↓↓↓↓↓↓↓
                    if (!getInputBuffer().parseHeaders()) {
                        openSocket = true;
                        readComplete = false;
                        break;
                    }
                    if (!disableUploadTimeout) {
                        setSocketTimeout(connectionUploadTimeout);
                    }
                }
            } catch (IOException e) {
              。。。。。。
            }

            if (!getErrorState().isError()) {
                // Setting up filters, and parse some request headers
                rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
                try {
                    prepareRequest();   // 校验和解析请求头中的属性
                } catch (Throwable t) {
                  。。。。。
                }
            }

            if (maxKeepAliveRequests == 1) {
                keepAlive = false;
            } else if (maxKeepAliveRequests > 0 &&
                    socketWrapper.decrementKeepAlive() <= 0) {
                keepAlive = false;
            }

            // Process the request in the adapter  调用适配器的 service 方法
            if (!getErrorState().isError()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    adapter.service(request, response); 
                    if(keepAlive && !getErrorState().isError() && (
                            response.getErrorException() != null ||
                                    (!isAsync() &&
                                    statusDropsConnection(response.getStatus())))) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                    setCometTimeouts(socketWrapper);
                } catch (InterruptedIOException e) {
                    。。。。。。。
                }
            }

            // Finish the handling of the request 完成请求的处理
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);

            if (!isAsync() && !comet) {
                if (getErrorState().isError()) {
                    getInputBuffer().setSwallowInput(false);
                } else {

                    checkExpectationAndResponseStatus();
                }
                endRequest();  //结束
            }
//。。。。。...
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);
        }
//。。。。。...
    }
```

ok，这里稍微说明一下，大概的处理时这个样子的：

* 从Socket中获取输入输出流
* 解析请求行和请求头
* 校验并解析请求头中的属性
* 调用适配器的service方法
* 请求处理结束

> ```markdown
> 还有说明一下的是hHttp 协议中的标准请求信息数据的格式： 
> * 请求行(request line)
> 例如GET /images/logo.gif HTTP/1.1，表示从/images目录下请求logo.gif这个文件。
> * 请求头(request header)，空行
> 例如Accept-Language: en
> * 其他消息体
> 请求行和标题必须以`<CR><LF>`作为结尾。空行内必须只有`<CR><LF>`而无其他空格。在 HTTP/1.1 协议中，所有的请求头，除 Host 外，都是可选的。 
> 请求行、请求头数据的格式具体看 Http 协议中的描述。所以在从输入流中读取到字节流数据之后必须按照请求行、请求头、消息体的顺序来解析。 
> 这里以请求行数据的解析为例，在 Http 协议中该行内容格式为：
> Request-Line = Method SP Request-URI SP HTTP-Version CRLF
> 即请求类型、要访问的资源（ URI ）以及使用的HTTP版本，中间以特殊字符空格来分隔，以`\r\n`字符结尾。
> ```

这里分析下一个，处理请求行的方法：`getInputBuffer().parseRequestLine(keptAlive)`：

实现类为`org.apache.coyote.http11.InternalInputBuffer `中的`parseRequestLine  `方法：

代码如下面这个样子的：↓↓↓↓↓↓↓↓

```java
 @Override
    public boolean parseRequestLine(boolean useAvailableDataOnly)
        throws IOException {
        int start = 0;
        byte chr = 0;
        do {
            if (pos >= lastValid) {
                if (!fill()) //这里调用当前类的fill方法
                    throw new EOFException(sm.getString("iib.eof.error"));
            }
            if (request.getStartTime() < 0) {
                request.setStartTime(System.currentTimeMillis());
            }
            chr = buf[pos++];
        } while ((chr == Constants.CR) || (chr == Constants.LF));
        pos--;
        start = pos;
        boolean space = false;
        
        while (!space) {
            // Read new bytes if needed
            if (pos >= lastValid) {
                if (!fill())
                    throw new EOFException(sm.getString("iib.eof.error"));
            }
            
            //根据请求头协议的格式，从中取出表示请求方法的字节数据并设置到内置实例变量 request
            //这里循环到行尾，把当前一行的参数设置到request中
            if (buf[pos] == Constants.SP || buf[pos] == Constants.HT) {
                space = true;
                request.method().setBytes(buf, start, pos - start);
            } else if (!HttpParser.isToken(buf[pos])) {
                throw new IllegalArgumentException(sm.getString("iib.invalidmethod"));
            }
            pos++;
        }

        //解析 method 和 uri 之间的空格字节 SP
        while (space) {
            // Read new bytes if needed
            if (pos >= lastValid) {
                if (!fill())
                    throw new EOFException(sm.getString("iib.eof.error"));
            }
            if (buf[pos] == Constants.SP || buf[pos] == Constants.HT) {
                pos++;
            } else {
                space = false;
            }
        }

        // Mark the current buffer position
        start = pos;
        int end = 0;
        int questionPos = -1;

        // Reading the URI
		//读取表示请求的 URI 的字节数据并放到 request 变量中
        boolean eol = false;
        while (!space) {
            // Read new bytes if needed
            if (pos >= lastValid) {
                if (!fill())
                    throw new EOFException(sm.getString("iib.eof.error"));
            }
            // Spec says single SP but it also says be tolerant of HT
            if (buf[pos] == Constants.SP || buf[pos] == Constants.HT) {
                space = true;
                end = pos;
            } else if ((buf[pos] == Constants.CR)
                       || (buf[pos] == Constants.LF)) {
                // HTTP/0.9 style request
                eol = true;
                space = true;
                end = pos;
            } else if ((buf[pos] == Constants.QUESTION) && (questionPos == -1)) {
                questionPos = pos;
            } else if (questionPos != -1 && !httpParser.isQueryRelaxed(buf[pos])) {
                throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget"));
            } else if (httpParser.isNotRequestTargetRelaxed(buf[pos])) {
                throw new IllegalArgumentException(sm.getString("iib.invalidRequestTarget"));
            }
            pos++;
        }
        //这里获取到参数结束位置，然后把字节流信息放到request中
        request.unparsedURI().setBytes(buf, start, end - start);

。。。

        // Mark the current buffer position
        start = pos;
        end = 0;

        //读取表示请求的 Http 协议版本的字节数据并放到 request 变量中
        while (!eol) {
            // Read new bytes if needed
            if (pos >= lastValid) {
                if (!fill())
                    throw new EOFException(sm.getString("iib.eof.error"));
            }

            if (buf[pos] == Constants.CR) {
                end = pos;
            } else if (buf[pos] == Constants.LF) {
                if (end == 0)
                    end = pos;
                eol = true;
            } else if (!HttpParser.isHttpProtocol(buf[pos])) {
                throw new IllegalArgumentException(sm.getString("iib.invalidHttpProtocol"));
            }
            pos++;
        }

        if ((end - start) > 0) {
            request.protocol().setBytes(buf, start, end - start);
        } else {
            request.protocol().setString("");
        }
        return true;
    }

    protected boolean fill(boolean block) throws IOException {
        int nRead = 0;
        if (parsingHeader) {
            if (lastValid == buf.length) {
                throw new IllegalArgumentException
                    (sm.getString("iib.requestheadertoolarge.error"));
            }
            nRead = inputStream.read(buf, pos, buf.length - lastValid);
            if (nRead > 0) {
                lastValid = pos + nRead;
            }
        } else {
            if (buf.length - end < 4500) {
                buf = new byte[buf.length];//从输入流中读取数据到缓冲区 buf
                end = 0;
            }
            pos = end;
            lastValid = pos;
            nRead = inputStream.read(buf, pos, buf.length - lastValid);
            if (nRead > 0) {
                lastValid = pos + nRead;
            }
        }
        return (nRead > 0);
    }
```

以上就是，根据 Http 协议解析请求行（ request line ）的代码实现部分 ，其他的解析请求找对应的方法看源码就行了，跟这个差不多的。

**【就是按照HTTP的协议，解析数据的各种参数之类的，然后设置到request里面去】**



---

##

