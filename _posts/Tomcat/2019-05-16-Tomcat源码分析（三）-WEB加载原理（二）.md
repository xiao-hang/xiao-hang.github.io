---
layout: post
#标题配置
title:   Tomcat 源码分析（三）-WEB加载原理（二）-WEB应用中的Listener、Filter、Servlet 的加载和调用
#时间配置
date:   2019-05-16 18:00:00 +0800
#大类配置
categories: 源码
#小类配置
tag: Tomcat
---

* content
{:toc}



## 三、WEB应用中的Listener、Filter、Servlet 的加载和调用

### web配置的关联

前文提到了，在Tomcat进行加载web.xml配置的时候，就是在`org.apache.catalina.deploy.WebXml`类的 configureContext 方法中，可以看到很多的setXX,和addXX的方法，把文件解析后表示Servlet、Listener、Filter 的配置信息都与表示web应用的Context对象关联起来。

代码在之前就有完整的，这里选一点点看看：

**configureContext 方法中 Servlet、Listener、Filter 的配置信息设置相关的调用代码：**

```java
......   //设置 Filter 相关配置信息的。
for (FilterDef filter : filters.values()) {  
    if (filter.getAsyncSupported() == null) {
        filter.setAsyncSupported("false");
    }
    context.addFilterDef(filter);
}
for (FilterMap filterMap : filterMaps) { 
    context.addFilterMap(filterMap);
}
......//给应用添加 Listener 的。
 for (String listener : listeners) {  
            context.addApplicationListener(listener);
 }
...... //设置 Servlet 的相关配置信息的
for (ServletDef servlet : servlets.values()) { 
    Wrapper wrapper = context.createWrapper();
	......
......
```

> 这些是web.xml中的相关配置的设置，【servvlet3的话还包括注解中的配置，这些配置是在ContextConfig类的webConfig方法中会合并】。

需要注意的是，这里配置的相关配置信息，**仅仅是把配置信息保存到Context的相应实例变量中，而真正的在请求中响应的 Servlet、Listener、Filter 的实例并没有构造出来**。这里的配置实例时其额封装类的实例：StandardWrapper、ApplicationListener、FilterDef、FilterMap。

### 真正响应实例的构建

在StandardContext也就是web应用被Host构建的时候，会发布事件，最终会调用解析加载web.xml的方法。然后，这里会把解析的配置封装类关联到StandardContext中去。

在配置好了之后，真正的处理实例的构建确实在别的地方：

**`org.apache.catalina.core.StandardContext`类的 startInternal 方法中：**

```java
protected synchronized void startInternal() throws LifecycleException {

    // Add missing components as necessary  添加缺失的必要组件
  .........

    // Initialize character set mapper
    getCharsetMapper();

    // Post work directory
    postWorkDirectory();

    // Validate required extensions 验证所需扩展
    boolean dependencyCheck = true;
    try {
        dependencyCheck = ExtensionValidator.validateApplication
            (getResources(), this);
    }
......

    if (!dependencyCheck) {
        // 如果依赖项检查失败，则不使应用程序可用
        ok = false;
    }
......
    // Standard container startup 标准容器启动
    // Binding thread
    ClassLoader oldCCL = bindThread();

    try {
        if (ok) {
            // Start our subordinate components, if any
            Loader loader = getLoaderInternal();
            if ((loader != null) && (loader instanceof Lifecycle))
                ((Lifecycle) loader).start();
......
            Cluster cluster = getClusterInternal();
            if ((cluster != null) && (cluster instanceof Lifecycle))
                ((Lifecycle) cluster).start();
            Realm realm = getRealmInternal();
            if ((realm != null) && (realm instanceof Lifecycle))
                ((Lifecycle) realm).start();
            DirContext resources = getResourcesInternal();
            if ((resources != null) && (resources instanceof Lifecycle))
                ((Lifecycle) resources).start();

            // Notify our interested LifecycleListeners  
            // ************  发布事件，这里会触发web.xml的解析   ***************
            fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);

            // Start our child containers, if not already started 启动子容器
            for (Container child : findChildren()) {
                if (!child.getState().isAvailable()) {
                    child.start();
                }
            }

            // Start the Valves in our pipeline (including the basic),启动管道中的阀门
            if (pipeline instanceof Lifecycle) {
                ((Lifecycle) pipeline).start();
            }

            // Acquire clustered manager 获取群集管理器??
            Manager contextManager = null;
            Manager manager = getManagerInternal();
          ......
              contextManager = new StandardManager();
            ...
               manager = contextManager;
......

            // Configure default manager if none was specified
           ......


    // We put the resources into the servlet context
    if (ok)
        getServletContext().setAttribute
            (Globals.RESOURCES_ATTR, getResources());
   ......

    if (ok ) {
        if (getInstanceManager() == null) {
            javax.naming.Context context = null;
            if (isUseNaming() && getNamingContextListener() != null) {
                context = getNamingContextListener().getEnvContext();
            }
            Map<String, Map<String, String>> injectionMap = buildInjectionMap(
                    getIgnoreAnnotations() ? new NamingResources(): getNamingResources());
            //**********设置实例管理器为 DefaultInstanceManager **********
            setInstanceManager(new DefaultInstanceManager(context,
                    injectionMap, this, this.getClass().getClassLoader()));
            getServletContext().setAttribute(
                    InstanceManager.class.getName(), getInstanceManager());
        }
    }

    try {
        // Create context attributes that will be required
        if (ok) {
            getServletContext().setAttribute(
                    JarScanner.class.getName(), getJarScanner());
        }

        // Set up the context init params 设置初始化参数
        mergeParameters();

        // Call ServletContainerInitializers
      ......

        // Configure and call application event listeners
        if (ok) {
            if (!listenerStart()) {  //*******就是这里配置调用Listener*******
               ......
            }
        }
......

        // Configure and call application filters
        if (ok) {
            if (!filterStart()) { //*******就是这里配置调用Filter*******
                log.error(sm.getString("standardContext.filterFail"));
                ok = false;
            }
        }

        // Load and initialize all "load on startup" servlets
        if (ok) {
            if (!loadOnStartup(findChildren())){  //*******就是这里配置调用Servlet *******
                log.error(sm.getString("standardContext.servletFail"));
                ok = false;
            }
        }
        // Start ContainerBackgroundProcessor thread
        super.threadStart();
    } finally {
        // Unbinding thread
        unbindThread(oldCCL);
    }
.............................................................
}
```

> 这里的代码很长，做了删减，反正只看得懂注释╮(╯_╰)╭

这里的初始化开始的方法中，打星号注释的是要关注的代码。这里里面：

* 发布了一个`CONFIGURE_START_EVENT`事件，就是前面所说的触发web.xml解析的地方。
* 然后之后设置了实例管理器为 DefaultInstanceManager（这个类在后面谈实例构造时会用到）。
* **再后面调用了listenerStart ，filterStart ，loadOnStartup 方法，这里即触发 Listener、Filter、Servlet 真正对象的构造。**

### 分析listenerStart 方法的-构造代码

代码还是很长的，删减一下只看Listenner 对象构造相关的代码：

```java
public class StandardContext extends ContainerBase
        implements Context, NotificationEmitter {
/** 配置实例化的应用程序事件侦听器集
 * Configure the set of instantiated application event listeners
 * for this Context. 
 */
public boolean listenerStart() {
.......
    ApplicationListener listeners[] = applicationListeners;
    Object results[] = new Object[listeners.length];
    boolean ok = true;
    for (int i = 0; i < results.length; i++) {
        ......
        try {
            ApplicationListener listener = listeners[i];
            results[i] = getInstanceManager().newInstance(
                    listener.getClassName());
            if (listener.isPluggabilityBlocked()) {
                noPluggabilityListeners.add(results[i]);
            }
        } catch (Throwable t) {
          ......
        }
    }
 ......
}
```

这里从Context对象中取出实例变量applicationListeners，这个变量在 web.xml 解析的时候设置的。然后通过：

```java
getInstanceManager().newInstance(listener.getClassName());
```

这里`getInstanceManager`所得到的对象就是`startInternal `方法里面设置的：`DefaultInstanceManager `对象。所以，这里的操作就是**DefaultInstanceManager 类的 newInstance 方法：**

```java
@Override
public Object newInstance(String className) throws IllegalAccessException,
InvocationTargetException, NamingException, InstantiationException,
ClassNotFoundException, IllegalArgumentException, NoSuchMethodException, SecurityException {
    Class<?> clazz = loadClassMaybePrivileged(className, classLoader);
    return newInstance(clazz.getDeclaredConstructor().newInstance(), clazz);
}
```

**这里就已经构建出来了，在web.xml中配置的Listener对象了。**

### 分析filterStart 方法的-Filter 的构建

> 这里的代码和实例化Listener的其实挺像的。。

```java
public boolean filterStart() {
    if (getLogger().isDebugEnabled())
        getLogger().debug("Starting filters");
    // Instantiate and record a FilterConfig for each defined filter
    boolean ok = true;
    synchronized (filterConfigs) {
        filterConfigs.clear();
        for (Entry<String, FilterDef> entry : filterDefs.entrySet()) {
            String name = entry.getKey();
            if (getLogger().isDebugEnabled())
                getLogger().debug(" Starting filter '" + name + "'");
            ApplicationFilterConfig filterConfig = null;
            try {
                filterConfig =
                    new ApplicationFilterConfig(this, entry.getValue());
                filterConfigs.put(name, filterConfig);
            } catch (Throwable t) {
                t = ExceptionUtils.unwrapInvocationTargetException(t);
                ExceptionUtils.handleThrowable(t);
                getLogger().error
                    (sm.getString("standardContext.filterStart", name), t);
                ok = false;
            }
        }
    }
    return (ok);
}
```

这段代码比较简单，就是取出web.xml解析的时候保存的filter配置信息的集合，进行处理。

这里的看一下15行：`new ApplicationFilterConfig(this, entry.getValue());`这里创建的新的实例。

```java
public final class ApplicationFilterConfig implements FilterConfig, Serializable {
ApplicationFilterConfig(Context context, FilterDef filterDef)
        throws ...... {
    super();
    this.context = context;
    this.filterDef = filterDef;
    // Allocate a new filter instance if necessary
    if (filterDef.getFilter() == null) {
        getFilter();
    } else {
        this.filter = filterDef.getFilter();
        getInstanceManager().newInstance(filter);
        initFilter();
    }
}
 //这里在默认情况下filterDef中是没有Filter对象的，所以会调用getFilter()方法：
Filter getFilter() throws...... {
        // Return the existing filter instance, if any
        if (this.filter != null)
            return (this.filter);

        // Identify the class loader we will be using
        String filterClass = filterDef.getFilterClass();
        this.filter = (Filter) getInstanceManager().newInstance(filterClass);  //<<--看这里
        initFilter();  //当然 这里会按照Servlet规范进行初始化
        return (this.filter);
    }
```

这里可以看到最后实例化的方法也是：` getInstanceManager().newInstance`。

> 这里每一个Filter就是一个ApplicationFilterConfig，内部有Filter的实例对象。

### 分析loadOnStartup方法-Servlet 的构建

> ????

```java
/**
* 加载并初始化中标记为“启动时加载”的所有servlet
* Web应用程序部署描述符。
 */
public boolean loadOnStartup(Container children[]) {

    // Collect "load on startup" servlets that need to be initialized
    TreeMap<Integer, ArrayList<Wrapper>> map =
        new TreeMap<Integer, ArrayList<Wrapper>>();
    for (int i = 0; i < children.length; i++) {
        Wrapper wrapper = (Wrapper) children[i];
        int loadOnStartup = wrapper.getLoadOnStartup();
        if (loadOnStartup < 0)
            continue;
        Integer key = Integer.valueOf(loadOnStartup);
        ArrayList<Wrapper> list = map.get(key);
        if (list == null) {
            list = new ArrayList<Wrapper>();
            map.put(key, list);
        }
        list.add(wrapper);
    }

    // Load the collected "load on startup" servlets
    for (ArrayList<Wrapper> list : map.values()) {
        for (Wrapper wrapper : list) {
            try {
                wrapper.load();
            } catch (ServletException e) {
                getLogger().error(sm.getString("standardContext.loadOnStartup.loadException",
                      getName(), wrapper.getName()), StandardWrapper.getRootCause(e));
                if(getComputedFailCtxIfServletStartFails()) {
                    return false;
                }
            }
        }
    }
    return true;
}
```

这里会对Servlet进行初始化实例操作代码是：wrapper.load(); 【StandardWrapper.load()】

需要注意的是，这里只针对配置了 load-on-startup 属性的 Servlet。

```java
public class StandardWrapper extends ContainerBase
    implements ServletConfig, Wrapper, NotificationEmitter {
@Override
public synchronized void load() throws ServletException {
    instance = loadServlet();

    if (!instanceInitialized) {
        initServlet(instance);
    }

    if (isJspServlet) {
        StringBuilder oname =
            new StringBuilder(MBeanUtils.getDomain(getParent()));

        oname.append(":type=JspMonitor,name=");
        oname.append(getName());
        oname.append(getWebModuleKeyProperties());

        try {
            jspMonitorON = new ObjectName(oname.toString());
            Registry.getRegistry(null, null)
                .registerComponent(instance, jspMonitorON, null);
        } catch( Exception ex ) {...... }
    }
}
//在上面的代码中，第5行，会调用下面的方法↓↓↓↓↓↓↓代码太多了，都删了 ╮(╯▽╰)╭  ↓↓↓↓↓↓↓↓↓
    public synchronized Servlet loadServlet() throws ServletException {
      ......
        Servlet servlet;
        try {
           ......  //要看的就是这下面的两行实例化的代码 ，跟Filter和Listener 差不多的
            InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
            try {
                servlet = (Servlet) instanceManager.newInstance(servletClass);
            } catch (ClassCastException e) {...... }
            ......
            initServlet(servlet);
         ......
        } finally {
           ......
        }
        return servlet;
    }
```

这里构造Servlet对象，与Filter类似。看看就好知道了

**需要注意的是：这里的加载只是针对配置了 load-on-startup 属性的 Servlet 而言，其它一般 Servlet 的加载和初始化会推迟到真正请求访问 web 应用而第一次调用该 Servlet 时**

### 请求时 相关Filter、Servlet的构建

> 在非配置load-on-startup 属性的 Servlet 而言，是不会再系统加载的时候创建具体的处理实例对象，依旧还只是个配置记录在Context中。真正的创建则是在第一次被请求的时候，才会实例化【这也是为什么有时候系统第一次访问会相对慢一点点了】

根据Tomcat一次请求的流程，可以知道请求会在容器Engine、Host、Context、Wrapper 各级组件中匹配，并且在他们的管道中流转。最终是会适配到一个StandardWrapper 的基础阀的-`org.apache.catalina.core.StandardWrapperValve`的 invoke 方法。

先在就来看看，请求匹配到最基础的StandardWrapper 组件的管道中，之后是如何处理的。

**看下StandardWrapperValve阀的invoke 方法：**

```java
final class StandardWrapperValve extends ValveBase {
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Initialize local variables we may need
    boolean unavailable = false;
    Throwable throwable = null;
    // This should be a Request attribute...
    long t1=System.currentTimeMillis();
    requestCount.incrementAndGet();
    StandardWrapper wrapper = (StandardWrapper) getContainer();
    Servlet servlet = null;
    Context context = (Context) wrapper.getParent();

    // Check for the application being marked unavailable 检查是否为不可用
    if (!context.getState().isAvailable()) {
      ......
    }

    // Check for the servlet being marked unavailable
    if (!unavailable && wrapper.isUnavailable()) {
        ...... unavailable = true;
    }

    // Allocate a servlet instance to process this request
    //*****分配一个servlet实例来处理这个请求 *****
    try {
        if (!unavailable) {
            servlet = wrapper.allocate();  //********这里是要关注的*******
            // 在allocate 方法中可能会调用 loadServlet() 方法,就是前边那个构建servlet的方法
        }
    } catch (UnavailableException e) {
        ......
    }

    // Identify if the request is Comet related now that the servlet has been allocated
    // 确定在分配servlet之后，请求是否与Comet相关
    boolean comet = false;
    if (servlet instanceof CometProcessor && Boolean.TRUE.equals(request.getAttribute(
            Globals.COMET_SUPPORTED_ATTR))) {
        comet = true;
        request.setComet(true);
    }
......
    // Create the filter chain for this request 
    ApplicationFilterFactory factory =
        ApplicationFilterFactory.getInstance();
     //这里↓↓↓↓会构造一个过滤器链（ filterChain ）用于执行这一次请求所经过的相应 Filter
    ApplicationFilterChain filterChain =
        factory.createFilterChain(request, wrapper, servlet);

    // Reset comet flag value after creating the filter chain
    request.setComet(false);

    // Call the filter chain for this request
    // NOTE: This also calls the servlet's service() method
    try {
        if ((servlet != null) && (filterChain != null)) {
            // Swallow output if needed
            if (context.getSwallowOutput()) {
                try {
                    SystemLogHandler.startCapture();
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else if (comet) {
                        filterChain.doFilterEvent(request.getEvent());
                        request.setComet(true);
                    } else {   
                        //********注意一下***********↓↓↓↓↓
                        filterChain.doFilter(request.getRequest(),
                                response.getResponse());
                    }
                } finally {.......}
            } else {
                if (request.isAsyncDispatching()) {
                    request.getAsyncContextInternal().doInternalDispatch();
                } else if (comet) {
                    request.setComet(true);
                    filterChain.doFilterEvent(request.getEvent());
                } else {
                    //********注意一下***********↓↓↓↓↓
                    filterChain.doFilter    
                        (request.getRequest(), response.getResponse());
                }
            }
        }
    } catch (ClientAbortException e) {......}

    // Release the filter chain (if any) for this request
    if (filterChain != null) {
        if (request.isComet()) {
            // If this is a Comet request, then the same chain will be used for the
            // processing of all subsequent events.
            filterChain.reuse();
        } else {
            filterChain.release();
        }
    }

    // Deallocate the allocated servlet instance 取消分配已分配的servlet实例
    try {
        if (servlet != null) {
            wrapper.deallocate(servlet);
        }
    } catch (Throwable e) {......}

    // If this servlet has been marked permanently unavailable,
    // unload it and release this instance
    try {
        if ((servlet != null) &&
            (wrapper.getAvailable() == Long.MAX_VALUE)) {
            wrapper.unload();
        }
    } catch (Throwable e) {......}
    long t2=System.currentTimeMillis();
    long time=t2-t1;
    processingTime += time;
    if( time > maxTime) maxTime=time;
    if( time < minTime) minTime=time;
}
```

在以上的代码中，需要注意的是第30行：

```java
 servlet = wrapper.allocate(); 
```

**这个`StandardWrapper .allocate()`的方法，在方法中会调用`instance = loadServlet();`这个方法，就是前面构建Servlet所调用的方法。**

**然后，在之后会构建filterChain-过滤器链，用于执行一次请求所经过的相应Filter。接着会链式调用filterChain 的doFilter方法。** 来看一下doFilter的具体实现：

```java
//final class ApplicationFilterChain implements FilterChain, CometFilterChain {
public void doFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {

    if( Globals.IS_SECURITY_ENABLED ) {
        final ServletRequest req = request;
        final ServletResponse res = response;
        try {
            java.security.AccessController.doPrivileged(
                new java.security.PrivilegedExceptionAction<Void>() {
                    @Override
                    public Void run() 
                        throws ServletException, IOException {
                        internalDoFilter(req,res);  //********
                        return null;
                    }
                }
            );
        } catch( PrivilegedActionException pe) {......}
    } else {
        internalDoFilter(request,response);//********
    }
}
//这里会调用到internalDoFilter方法  就在下面
private void internalDoFilter(ServletRequest request, 
                                  ServletResponse response)
        throws IOException, ServletException {

        // Call the next filter if there is one
        if (pos < n) {  //n表示：表示过滤器链中所有的过滤器 pos：表示当前要执行的过滤器
            ApplicationFilterConfig filterConfig = filters[pos++];
            Filter filter = null;
            try {
                filter = filterConfig.getFilter();
                support.fireInstanceEvent(InstanceEvent.BEFORE_FILTER_EVENT,
                                          filter, request, response);
                if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                        filterConfig.getFilterDef().getAsyncSupported())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                            Boolean.FALSE);
                }
                if( Globals.IS_SECURITY_ENABLED ) {
                    final ServletRequest req = request;
                    final ServletResponse res = response;
                    Principal principal = 
                        ((HttpServletRequest) req).getUserPrincipal();

                    Object[] args = new Object[]{req, res, this};
                    SecurityUtil.doAsPrivilege
                        ("doFilter", filter, classType, args, principal);
                } else {  
                    filter.doFilter(request, response, this);  //***就是这个***
                }

                support.fireInstanceEvent(InstanceEvent.AFTER_FILTER_EVENT,
                                          filter, request, response);
            } catch (IOException e) {......}
            return;
        }
		// 我们脱离了链的末端——  ****调用servlet实例****
        // We fell off the end of the chain -- call the servlet instance
        try {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(request);
                lastServicedResponse.set(response);
            }

            support.fireInstanceEvent(InstanceEvent.BEFORE_SERVICE_EVENT,
                                      servlet, request, response);
            if (request.isAsyncSupported()
                    && !support.getWrapper().isAsyncSupported()) {
                request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                        Boolean.FALSE);
            }
            // Use potentially wrapped request from this point
            if ((request instanceof HttpServletRequest) &&
                (response instanceof HttpServletResponse)) {
                    
                if( Globals.IS_SECURITY_ENABLED ) {
                    final ServletRequest req = request;
                    final ServletResponse res = response;
                    Principal principal = 
                        ((HttpServletRequest) req).getUserPrincipal();
                    Object[] args = new Object[]{req, res};
                    SecurityUtil.doAsPrivilege("service",
                                               servlet,
                                               classTypeUsedInService, 
                                               args,
                                               principal);   
                } else {  
                    servlet.service(request, response); //这里就执行具体方法了
                }
            } else {
                servlet.service(request, response);//这里就执行具体方法了
            }
            support.fireInstanceEvent(InstanceEvent.AFTER_SERVICE_EVENT,
                                      servlet, request, response);
        } catch (IOException e) {......} finally {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(null);
                lastServicedResponse.set(null);
            }
        }
    }
```

这里的**internalDoFilter**方法，在52行的地方会执行：

```java
 filter.doFilter(request, response, this);
```

并且在一般的过滤器的使用中，最后都会有这一句：

```java
FilterChain.doFilter(request, response);
```

这里就**回到了filterChain 的 doFilter 方法**，就形成了递归调用。

**然后：filterChain 对象内部的 pos 是不断加的，所以假如过滤器链中的各个 Filter 的 doFilter 方法都执行完之后,就会执行到调用servlet的service 方法的地方** 【第60行第地方开始】

### 结束

到这里，请求就在经过ListenerStart，Filter 调用到了具体的Servlet方法 ，就是web应用里面的Controller层的业务了。

> 终于了解到请求走到了自己写代码的地方了  ^_^

---

2019-05-13 小杭  

---
