# Tomcat 源码分析-请求分析（二）

[TOC]

## 三、请求与容器中具体组件的匹配

> 到，前一篇为止，已经分析到了`org.apache.coyote.http11.AbstractHttp11Processor`类 process 方法 ，了解到了请求从Socket的IO流中取出字节流，很久http协议将字节流组装到Tomcat的内部对象`org.apache.coyote.Request `中的相关属性中。

接下来这里，要分析了解的，就是：Tomcat的内部请求对象怎么从 Connector 到 Engine 到 Host 到 Context 最后到 Servlet 的过程的。

### 开始进行内部传递处理的地方

```java
adapter.service(request, response);
```

> 这一行代码就是在AbstractHttp11Processor的process 方法中，使用适配器处理的地方。【是在处理socket的数据，解析请求头之后那里】

这里的adapter对象，是在Http11Processor 创建的时候设置的：

```java
//org.apache.coyote.http11.Http11Protocol.Http11ConnectionHandler类的 createProcessor 方法
 protected Http11Protocol proto;
 protected Http11Processor createProcessor() {
            Http11Processor processor = new Http11Processor(
                    proto.getMaxHttpHeaderSize(), proto.getRejectIllegalHeaderName(),
                    (JIoEndpoint)proto.endpoint, proto.getMaxTrailerSize(),
                    proto.getAllowedTrailerHeadersAsSet(), proto.getMaxExtensionSize(),
                    proto.getMaxSwallowSize(), proto.getRelaxedPathChars(),
                    proto.getRelaxedQueryChars());
            processor.setAdapter(proto.adapter); //可以看到在这里进行的设置
.....
            register(processor);
            return processor;
        }
```

这里可以看的出来，adapter 使用的是Http11Protocol的adapter 变量，

变量的创建则是在Connector类的初始化方法initInternal 中：（配置文件Connector节点的初始化过程中）

```java
//org.apache.catalina.connector;
@Override
protected void initInternal() throws LifecycleException {
    super.initInternal();
    // Initialize adapter 在这里对adapter进行初始化，使用的是CoyoteAdapter类
    adapter = new CoyoteAdapter(this);
    protocolHandler.setAdapter(adapter);
	//上面这种双向绑定对象的方式在Tomcat中相当的常见啊 _(¦3」∠)_
    // Make sure parseBodyMethodsSet has a default
    if( null == parseBodyMethodsSet ) {
        setParseBodyMethods(getParseBodyMethods());
    }
  ......
    try {
        protocolHandler.init();
    } catch (Exception e) {
    .......
    }
    // Initialize mapper listener
    mapperListener.init();
}
```

### 实际执行的处理方法

这里可知道了`adapter.service(request, response)`所调用的方法实际执行的是`org.apache.catalina.connector.CoyoteAdapter`类的 service 方法。

部分关键的源码分析：

```java
public void service(org.apache.coyote.Request req,
                        org.apache.coyote.Response res)
        throws Exception {
		//	这里的时候，就已经把参数req 从org.apache.coyote.Request 转换为 org.apache.catalina.connector.Request，这个才是真正在Tomcat容器流转的对象，也是我们平常使用的。
    // 这里的getNote(1)基本就是req差不多的东东，但是少了一些 headers，cookies之类的信息
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
			//这里开始 就是给创建的Request 和 Response 对象设置各种的值和传递的参数
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);
            // Link objects
            request.setResponse(response);
            response.setRequest(request);
            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);
            // Set query string encoding
            req.getParameters().setQueryStringEncoding
                (connector.getURIEncoding());
        }
        if (connector.getXpoweredBy()) {
            response.addHeader("X-Powered-By", POWERED_BY);
        }
        boolean comet = false;
        boolean async = false;
        boolean postParseSuccess = false;
        try {
            // Parse and set Catalina and configuration specific
            // request parameters 分析并设置catalina和特定于配置的请求参数
            // 这里是一个重点，postParseRequest方法会吧req对象里面的一些变量 host、context、warp 信息
            // 设置到新的request，response中去，已达到在其中流转。
            req.getRequestProcessor().setWorkerThreadName(Thread.currentThread().getName());
            postParseSuccess = postParseRequest(req, request, res, response);
            //*****************↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
            if (postParseSuccess) {
                //check valves if we support async  
request.setAsyncSupported(connector.getService().getContainer().getPipeline().isAsyncSupported());
                // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ 这里是一个很重要的地方，进行了数据的传递流转
                connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
                 //这个地方在下面说明的 阀机制原理的时候会分析到这里↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                if (request.isComet()) {
                    if (!response.isClosed() && !response.isError()) {
                        if (request.getAvailable() || (request.getContentLength() > 0 && (!request.isParametersParsed()))) {
                            // Invoke a read event right away if there are available bytes
                            if (event(req, res, SocketStatus.OPEN_READ)) {
                                comet = true;
                                res.action(ActionCode.COMET_BEGIN, null);
                            } else {
                                return;
                            }
                        } else {
                            comet = true;
                            res.action(ActionCode.COMET_BEGIN, null);
                        }
                    } else {
                        // Clear the filter chain, as otherwise it will not be reset elsewhere
                        // since this is a Comet request
                        request.setFilterChain(null);
                    }
                }
            }
```

**service 方法的主要功能就是，创建了最终我们使用的Request和Response，并对其请求传递来的各种数据进行设置到新的对象里面去**，包括host、context、warp 信息(这些请求接下来要在Tomcat中流转的信息)。

这个是在第37行的`postParseRequest(req, request, res, response)`方法中实现的。

### 分析一下对其参数的匹配过程

具体的，就是来看一下这个postParseRequest方法：

```java
/** Parse additional request parameters.分析其他请求参数，设置了各种参数到新的对象里面去*/
    protected boolean postParseRequest(org.apache.coyote.Request req,
                                       Request request,
                                       org.apache.coyote.Response res,
                                       Response response)
            throws Exception {

        if (! req.scheme().isNull()) {
            request.setSecure(req.scheme().equals("https"));
        } else {
            //使用连接器方案和安全配置(defaults to "http" and false respectively)
            req.scheme().setString(connector.getScheme());
            request.setSecure(connector.getSecure());
        }

        // 配置端口 未显式设置。根据方案使用默认端口 代理端口 没配置这段代码就跳过了
        String proxyName = connector.getProxyName();
        int proxyPort = connector.getProxyPort();
        if (proxyPort != 0) {
            req.setServerPort(proxyPort);
        } else if (req.getServerPort() == -1) {
            if (req.scheme().equals("https")) {
                req.setServerPort(443);
            } else {
                req.setServerPort(80);
            }
        }
        if (proxyName != null) {
            req.serverName().setString(proxyName);
        }

        // Copy the raw URI to the decodedURI
        MessageBytes decodedURI = req.decodedURI();
        decodedURI.duplicate(req.requestURI());
        parsePathParameters(req, request);

        // 这里看着是做一些URI的校验
        try {
            req.getURLDecoder().convert(decodedURI, false);
        } catch (IOException ioe) {
            res.setStatus(400);
            res.setMessage("Invalid URI: " + ioe.getMessage());
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }
        if (!normalize(req.decodedURI())) {
            res.setStatus(400);
            res.setMessage("Invalid URI");
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }
        convertURI(decodedURI, request);
        if (!checkNormalize(req.decodedURI())) {
            res.setStatus(400);
            res.setMessage("Invalid URI character encoding");
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }

        // Request mapping.
        MessageBytes serverName;
        if (connector.getUseIPVHosts()) {
            serverName = req.localName();
            if (serverName.isNull()) {
                res.action(ActionCode.REQ_LOCAL_NAME_ATTRIBUTE, null);
            }
        } else {
            serverName = req.serverName();
        }
        if (request.isAsyncStarted()) {
            request.getMappingData().recycle();
        }

        String version = null;
        Context versionContext = null;
        boolean mapRequired = true;

        while (mapRequired) {
            // wrapper，contxt,(request原本是空的) - 这里会做映射
            // 这里的map方法会对request.getMappingData()->MappingData 按请求的信息设置成员变量 host、context、warp 的信息，然后再把本身的 MappingData 的值对应设置到request；
            // 这里处理的 host、context、warp 的信息就确定了请求之后的处理方向了
            connector.getMapper().map(serverName, decodedURI, version,
                                      request.getMappingData());
            request.setContext((Context) request.getMappingData().context);
            request.setWrapper((Wrapper) request.getMappingData().wrapper);

// If there is no context at this point, it is likely no ROOT context has been deployed
            // 做一些请求数据的校验
            if (request.getContext() == null) {
                res.setStatus(404);
                res.setMessage("Not found");
                // No context, so use host
                Host host = request.getHost();
                // Make sure there is a host (might not be during shutdown)
                if (host != null) {
                    host.logAccess(request, response, 0, true);
                }
                return false;
            }

            // sessionID的解析
            String sessionID;
            if (request.getServletContext().getEffectiveSessionTrackingModes()
                    .contains(SessionTrackingMode.URL)) {

                // Get the session ID if there was one
                sessionID = request.getPathParameter(
                        SessionConfig.getSessionUriParamName(
                                request.getContext()));
                if (sessionID != null) {
                    request.setRequestedSessionId(sessionID);
                    request.setRequestedSessionURL(true);
                }
            }

            // Look for session ID in cookies and SSL session
            parseSessionCookiesId(req, request);
            parseSessionSslId(request);

            sessionID = request.getRequestedSessionId();
            
。。还有一堆代码 。。就不看了。。反正我是看不懂。。最多看个大概意思。。设置参数，校验下参数之类的。。
```

**这里对上面的代码稍微说明一下：**

参数匹配，在从原来的req复制过来之后，对很多的参数进行了校验判断。

其中，Host、Context、Wrapper 参数的设置尤其的重（在85-88行），这里的map方法在设置MappingData中的Host、Context、Wrapper信息的时候，是做了匹配的，这些信息表示的就是本次的请求之后要处理的对象了。

在Tomcat 中，Host、Context、Wrapper为一对多的关系，在请求进来的时候，把他们用Map维护到一个Connector 中，确保了一个请求，就指定一个web应用进行处理了。

> 这里的Map是怎么进行匹配的，我也不懂了╮(╯_╰)╭ 我看着应该是根据URI地址进行匹配的

### 匹配完成了

到这里，容器使用的Request，Response 对象已经重新创建好了，并且已经对一些参数进行设置和复制。

同时，是用Map的方法，对Context，Wrapper 参数的值做了对应的匹配，设置。

之后的处理流转就按照配置的参数进行了。



---

---



##  四、Tomcat 7 阀机制原理

这里的阀机制，就是数据（request）在Tomcat组件之间传递使用的东东。【类似阀门一样的通道的东西】

先看下这个图，Tomcat内的组件数据流转的图↓↓↓↓↓

![img](https://user-gold-cdn.xitu.io/2018/1/31/1614b97971ecddc5?imageView2/0/w/1280/h/960/format/jpeg/ignore-error/1)

> 这个图基本上已经画出了整个的过程，在 Connector 接收到一次连接并转化成请求（ Request ）后，会将请求传递到 Engine 的管道（ Pipeline ）的阀（ ValveA ）中。请求在 Engine 的管道中最终会传递到 Engine Valve 这个阀中。接着请求会从 Engine Valve 传递到一个 Host 的管道中，在该管道中最后传递到 Host Valve 这个阀里。接着从 Host Valve 传递到一个 Context 的管道中，在该管道中最后传递到 Context Valve 中。接下来请求会传递到 Wrapper C 内的管道所包含的阀 Wrapper Valve 中，在这里会经过一个过滤器链（ Filter Chain ），最终送到一个 Servlet 中。

这里来分析一下，这里面的管道（ Pipeline ）和阀（ Valve ）的操作【流转看上面图的箭头就可以了】

### 管道和阀 初始化和初次调用

这里管道来流转数据，出现的地方是在：承接上个文章中分析到的CoyoteAdapter类的service 方法中：【44行】

```java
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

这里就开始分析一下这一行的代码：

**connector.getService()：**获取当前的 connector 关联的 **Service 组件**

> 这里默认的对象是：`org.apache.catalina.core.StandardService`的对象。

**getContainer()：**获得的就是里面Engine容器的东东了

> 这里返回的是：`org.apache.catalina.core.StandardEngine`的对象。
>
> 这是在Server组件初始化的时候创建的：
>
> ```java
> digester.addRuleSet(new EngineRuleSet("Server/Service/"));
> ```
>
> 在 EngineRuleSet 类的 addRuleInstances 方法中的这一段代码:
>
> ```java
> public void addRuleInstances(Digester digester) {
> 
>     digester.addObjectCreate(prefix + "Engine",
>                              "org.apache.catalina.core.StandardEngine",
>                              "className");
>     digester.addSetProperties(prefix + "Engine");
>     digester.addRule(prefix + "Engine",
>                      new LifecycleListenerRule
>                      ("org.apache.catalina.startup.EngineConfig",
>                       "engineConfigClass"));
>     digester.addSetNext(prefix + "Engine",
>                         "setContainer",   //已Container方法设置了StandardEngine对象
>                         "org.apache.catalina.Container");
> ```
>
> 所以这里获取的是：org.apache.catalina.core.StandardEngine对象。

**getPipeline()：** StandardEngine 类的pipeline 变量

> 这里就是StandardEngine对象的getPipeline()方法，Pipeline 成员变量。
>
> 变量定义在其父类：ContainerBase 类中被定义：
>
> ```java
>  protected Pipeline pipeline = new StandardPipeline(this);
> ```
>
> 所以 ，这里返回的是`StandardPipeline`类的对象。

这里返回的StandardPipeline就是管道了，之后会详细分析一下的。

**getFirst().invoke(request, response)：**这个就是调用管道里面的阀，并执行内部方法流转数据了。



### 分析管道和阀的概念和实现

**管道Pipeline：**所谓的管道就是实现了`org.apache.catalina.Pipeline`这个接口，在上面的`new StandardPipeline(this);`这个代码的时候创建的。

> 这里的Pipeline接口中，最重要的就是
>
> basic：基础阀（通过 getBasic、setBasic 方法调用）
>
> first：第一个阀 
>
> vaules:普通阀（通过 addValve、removeValve 调用）
>
> > 这里需要细说一下**addValve方法**：
> >
> > ```java
> > public void addValve(Valve valve) {
> >     // Validate that we can add this Valve
> >     if (valve instanceof Contained)
> >         ((Contained) valve).setContainer(this.container);
> >     // Start the new component if necessary
> > 。。。。。
> >     //主要的就是下面这个：将此阀添加到与此管道关联的集合中
> >     if (first == null) {
> >         first = valve;
> >         valve.setNext(basic);
> >     } else {
> >         Valve current = first;
> >         while (current != null) {
> >             if (current.getNext() == basic) {
> >                 current.setNext(valve);
> >                 valve.setNext(basic);
> >                 break;
> >             }
> >             current = current.getNext();
> >         }
> >     }
> >     container.fireContainerEvent(Container.ADD_VALVE_EVENT, valve);
> > }
> > ```
> >
> > 这里的代码，会给管道添加一个普通阀的时候，已【first】和【basic】作为头尾，维护一个链表结构。

**阀Valve：**实现`org.apache.catalina.Valve`这个接口。

> 接口中主要的定义，就是setNext、getNext、invoke 这三个方法；
>
> 通过setNext设置该阀的下一阀，通过 getNext 返回该阀的下一个阀的引用，**invoke 方法则执行该阀内部自定义的请求处理代码。**
>
> 在invoke 方法中通常都会有这段代码：`getNext().invoke(request, response);`,这样就会执行所有的阀了。



### 具体的数据流转

+ **这里从上面分析的getPipeline()返回StandardPipeline类开始；**

这个类的初始化时在getContainer()返回的**StandardEngine**类的初始化方法中定义的：

```java
public class StandardEngine extends ContainerBase implements Engine {

    public StandardEngine() {
        super();
        pipeline.setBasic(new StandardEngineValve());   //这里插入了基础阀
        /* Set the jmvRoute using the system property jvmRoute */
        try {
            setJvmRoute(System.getProperty("jvmRoute"));
        } catch(Exception ex) {
            log.warn(sm.getString("standardEngine.jvmRouteFail"));
        }
        // By default, the engine will hold the reloading thread
        backgroundProcessorDelay = 10;
    }
    。。。。。。
```

第5行，设置了基础阀。所以

```java
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)
```

会执行的是 **StandardEngineValve**类的invoke方法。

+ **StandardEngineValve.invoke()方法：**

```java
 final class StandardEngineValve    extends ValveBase {
     @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Select the Host to be used for this Request
        Host host = request.getHost();
        if (host == null) {
            response.sendError
                (HttpServletResponse.SC_BAD_REQUEST,
                 sm.getString("standardEngine.noHost", 
                              request.getServerName()));
            return;
        }
        if (request.isAsyncSupported()) {
            request.setAsyncSupported(host.getPipeline().isAsyncSupported());
        }

        // Ask this Host to process this request 对就是这里获取下一个的容器来流转数据
        host.getPipeline().getFirst().invoke(request, response);
    }
     。。。。。。
```

这个从request中获取到Host对象，【默认是：`org.apache.catalina.core.StandardHost`对象】。然后就是第20行，进行获取管道以及执行阀的**invoke()**方法;

+ **StandardHost类的通道的阀的invoke()方法：**

StandardHost 的构造方法的实现：

```java
public StandardHost() {
    super();
    pipeline.setBasic(new StandardHostValve());
}
```

**StandardHostValve类的invoke()方法：**

```java
/**选择适当的子上下文来处理此请求，
基于指定的请求URI。如果没有匹配的上下文可以
如果找到，则返回相应的HTTP错误。**/
public final void invoke(Request request, Response response)
        throws IOException, ServletException {
        // Select the Context to be used for this Request
        Context context = request.getContext();  //获取到Context 组件对象
       //。。。一些判断。。。。。
        if (asyncAtStart || context.fireRequestInitEvent(request.getRequest())) {
            try {
                if (!asyncAtStart || asyncDispatching) {
                    //下↓↓↓↓↓↓↓↓面这里看看就可以了
                    context.getPipeline().getFirst().invoke(request, response);
                    //↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
                } else {
                    if (!response.isErrorReportRequired()) {
                        throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
                    }
                }
            } catch (Throwable t) {
             // 。。。。。。。。
                }
            }
           // 。。。。。。。。
            //现在请求/响应对回到容器下
            //控制提升悬架，以便错误处理可以
            //完成和/或容器可以刷新任何剩余数据
    }
```

这里可以看到Host【StandardHostValve】中流转，接下来会流转到Context，也就是`org.apache.catalina.core.StandardContext`类中的管道去。

+ **StandardContext类的通道的阀的invoke()方法：**

管道在StandardContext构建的时候创建：

```java
    public StandardContext() {
        super();
        pipeline.setBasic(new StandardContextValve()); // 就是这个 设置了当前的管道
        broadcaster = new NotificationBroadcasterSupport();
        if (!Globals.STRICT_SERVLET_COMPLIANCE) {
            resourceOnlyServlets.add("jsp");
        }
    }
```

这里的阀是：StandardContextValve 

```java
  public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        MessageBytes requestPathMB = request.getRequestPathMB();
        if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/META-INF"))
                || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
        Wrapper wrapper = request.getWrapper();
        if (wrapper == null || wrapper.isUnavailable()) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
        try {
            response.sendAcknowledgement();
        } catch (IOException ioe) {
        }

        if (request.isAsyncSupported()) {
            request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
        }
        wrapper.getPipeline().getFirst().invoke(request, response);  //要看的就只是这个
    }
```

这里，基础阀的invoke方法：是调用wrapper容器的管道进行数据流转。

wrapper 对象默认是`org.apache.catalina.core.StandardWrapper`类的实例

+ **StandardWrapper类的通道的阀的invoke()方法：**

设置基础阀：

```java
    public StandardWrapper() {
        super();
        swValve=new StandardWrapperValve();
        pipeline.setBasic(swValve);
        broadcaster = new NotificationBroadcasterSupport();
    }
```

然后就是StandardWrapperValve这个阀的invoke 方法；

```java
这个代码 戝长。。。。。不去copy了  //╮(╯_╰)╭#\\
    它的方法备注是酱紫的：
    
Invoke the servlet we are managing, respecting the rules regarding
servlet lifecycle and SingleThreadModel support.
调用我们正在管理的servlet，遵守有关
servlet生命周期和singlethreadmodel支持。
```

**意思就是：**
**在这里最终会调用请求的 url 所匹配的 Servlet 相关过滤器（ filter ）的 doFilter 方法及该 Servlet 的 service 方法（这段实现都是在过滤器链 ApplicationFilterChain 类的 doFilter 方法中）**

+ 数据流转结束

到这里，已经到了Tomcat 处理组件的最里层了。Wrapper 是对一个 Servlet 的包装，所以他的基础阀内部调用的是过滤链doFiter方法 和 Servlet 的 service  方法；



### 关于数据内部传递小结

 请求在service组件，完成请求分析组装成内部对象request和response之后，就根据组件之间的管道和阀进行数据传递和处理链的传递。

request对象在组装的时候已经适配了对应的组件，在《请求与容器中具体组件的匹配》的时候已经说明。组件内部的管道在初始化之后，会设置基础阀--基础阀的invoke方法会调用下一个组件的管道，`xxxxxxxx.getPipeline().getFirst().invoke(request, response);`进行数据处理的传递。

> 处理链的顺序是被固定的。
>
> 管道的处理会从First阀一直执行到基础阀。

这样子，数据和处理链就会一直传递处理到Wrapper 组件，也就是具体的一个Servlet 。

到这里Socket连接请求在Tomcat容器内的处理、传递就完成了。

> 之后的的数据和处理就到了 web 应用里面了

### 参考资料

* [Tomcat 7 的一次请求分析（一）处理线程的产生](http://www.iocoder.cn/Tomcat/yuliu/A-request-analysis-1-process-the-generation-of-threads/)
* [Tomcat 7 的一次请求分析（二）Socket 转换成内部请求对象](http://www.iocoder.cn/Tomcat/yuliu/A-request-analysis-2-Socket-is-converted-to-an-internal-request-object/)
* [Tomcat 7 的一次请求分析（三）请求与容器中具体组件的匹配](http://www.iocoder.cn/Tomcat/yuliu/A-request-analysis-3-The-request-matches-specific-components-in-the-container/)
* [Tomcat 7 的一次请求分析（四）Tomcat 7 阀机制原理](http://www.iocoder.cn/Tomcat/yuliu/A-request-analysis-4-Tomcat-7-valve-mechanism-principle/)



---

小杭  2019-04-18  

终于看完这一部分了  _(:з」∠)_  

