# Tomcat 源码分析（三）-WEB加载原理（一）

[TOC]

---

### 简要说明

之前的学习，是从Tomcat的启动脚本，到 `server.xml`解析,然后分析了一个请求的访问数据流转。

到此，只是分析了流转用到的组件的阀之类的方法，并没有分析说明组件的创建，尤其是Context这个组件，对应的就是我们的WEB应用。

然后，这份的学习就是：接着请求流转的具体到WEB应用的分析

> 可以和数据流转整合一起理解请求到我们代码中的流程

* 包括WEB的加载，解析web.xml，记载监听拦截器服务方法等

> 这里才是我最想知道的东西-容器是怎么个加载我的WEB应用的 (:3[」_]

## 一、Context 的构建，发布加载WEB应用事件

### 介绍

Tomcat 中启动的时候，默认就会有好几个线程，其中`main`、`http-bio-8080-Acceptor-0`、`http-bio-8080-AsyncTimeout`、`ajp-bio-8009-Acceptor-0`、`ajp-bio-8009-AsyncTimeout`，线程已经说明过了【是做为监听和守护进程的存在】。然后，这个还有一个线程：`ContainerBackgroundProcessor[StandardEngine[Catalina]]` ，这个线程直接就是服务的线程，接下来分析的就是它了。。。

### 线程的创建

说明从这个线程开始。

Tomcat7 中默认的容器（ StandardEngine、StandardHost、StandardContext、StandardWrapper ）这些，都会继承一个父类： `org.apache.catalina.core.ContainerBase`。在Tomcat组件启动的时候，会调用自己内部的`startInternal`方法，而这个方法中，会调用父类的方法：`super.startInternal();`。**比如StandardEngine的方法：**

```java
protected synchronized void startInternal() throws LifecycleException {
    if(log.isInfoEnabled())
        log.info( "Starting Servlet Engine: " + ServerInfo.getServerInfo());
    super.startInternal();   //←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
}
```

**这里会调用父类的方法，也就是：ContainerBase.startInternal()：**

```java
//启动该组件并实现需求    
protected synchronized void startInternal() throws LifecycleException {

        // Start our subordinate components, if any
        Loader loader = getLoaderInternal();
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();
        logger = null;
        getLogger();
        Manager manager = getManagerInternal();
        if ((manager != null) && (manager instanceof Lifecycle))
            ((Lifecycle) manager).start();
        Cluster cluster = getClusterInternal();
        ........

        // Start our child containers, if any
         .........
        // Start the Valves in our pipeline (including the basic), if any
    	//这里启动了管道中的阀 包括基础阀
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).start();
        }
    //↓↓↓↓这里发布了一个STARTING的事件，这个的作用后面点会说明↓↓↓↓↓
        setState(LifecycleState.STARTING);  
        // ↓↓↓↓↓↓↓↓↓↓↓ Start our thread  这里开始了一个线程 ****↓↓↓↓↓
        threadStart();
    }
```

**然后，看看这个启动线程的方法：`threadStart();`**

```java
/**
 * Start the background thread that will periodically check for
 * session timeouts. 启动将定期检查会话超时的后台线程。
 */
protected void threadStart() {
    if (thread != null)
        return;
    if (backgroundProcessorDelay <= 0)
        return;

    threadDone = false;
    String threadName = "ContainerBackgroundProcessor[" + toString() + "]";
    thread = new Thread(new ContainerBackgroundProcessor(), threadName);
    thread.setDaemon(true);
    thread.start();
}
```

> **说明一下**，threadName 这里就是现成的名字，而toString（）方法  ：以`org.apache.catalina.core.StandardEngine`为例：
>
> ```java
> public String toString() {
>     StringBuilder sb = new StringBuilder("StandardEngine[");
>     sb.append(getName());
>     sb.append("]");
>     return (sb.toString());
> }
> ```
>
> **所以，这个线程的启动以及名字已经找到出处了。**

这个`threadStar()t`在所有组件初始化的时候多会调用到,而这里只会存在一个线程，

分析一下：

```java
/* 默认值 ： */ thread = null；backgroundProcessorDelay= -1；
```

然而在几个的组件方法中，只有`StandardEngine`的构造方法对`backgroundProcessorDelay`进行修改：

```java
public StandardEngine() {
    super();
    pipeline.setBasic(new StandardEngineValve());
    try {
        setJvmRoute(System.getProperty("jvmRoute"));
	。。。。。
    // By default, the engine will hold the reloading thread
    backgroundProcessorDelay = 10;  //←←←←←←←←←←←←←←←←←←←←←←
}
```

**SO，在Tomcat 解析xml到一个Engine节点的时候就会产生一个后台处理线程。**

### 线程的处理事务

这里接下来，就是来分析一下这个线程具体是干什么的了。

```java
new Thread(new ContainerBackgroundProcessor(), threadName);
```

这里可以看到，将会启动的线程是内部类`ContainerBackgroundProcessor.run()`;

```java
/** 私有线程类，用于在固定延迟后调用此容器及其子级的BackgroundProcess方法 
 * Private thread class to invoke the backgroundProcess method
 * of this container and its children after a fixed delay.
 */
protected class ContainerBackgroundProcessor implements Runnable {

    @Override
    public void run() {
        Throwable t = null;
        String unexpectedDeathMessage = sm.getString(
                "containerBase.backgroundProcess.unexpectedThreadDeath",
                Thread.currentThread().getName());
        try {
            while (!threadDone) {
                try {
                    Thread.sleep(backgroundProcessorDelay * 1000L);
                } catch (InterruptedException e) {
                    // Ignore
                }
                if (!threadDone) {
                    Container parent = (Container) getMappingObject();
                    ClassLoader cl =
                        Thread.currentThread().getContextClassLoader();
                    if (parent.getLoader() != null) {
                        cl = parent.getLoader().getClassLoader();
                    }
                    processChildren(parent, cl);   //这里就是为了定时的调用这个方法
                }
            }
        } catch (RuntimeException e) {
          .........
        }
    }

    protected void processChildren(Container container, ClassLoader cl) {
        try {
            if (container.getLoader() != null) {
                Thread.currentThread().setContextClassLoader
                    (container.getLoader().getClassLoader());
            }
            //调用本身容器的backgroundProcess方法
            container.backgroundProcess();
        } catch (Throwable t) {
         ......
        }
        //取出并调用容器的所以子容器的processChildren 方法
        Container[] children = container.findChildren();
        for (int i = 0; i < children.length; i++) {
            if (children[i].getBackgroundProcessorDelay() <= 0) {
                processChildren(children[i], cl);
            }
        }
    }
}
```

**总结来说：归结起来这个线程的实现就是定期通过递归的方式调用当前容器及其所有子容器的 backgroundProcess 方法。**

**解析下这个backgroundProcess方法：**而这个 backgroundProcess 方法在 ContainerBase 内部已经给出了实现：

```java
@Override
public void backgroundProcess() {

    if (!getState().isAvailable())
        return;

    Cluster cluster = getClusterInternal();
    if (cluster != null) {
        try {
            cluster.backgroundProcess();
        } catch (Exception e) {
            log.warn(sm.getString("containerBase.backgroundProcess.cluster", cluster), e);
        }
    }
    //删减后的↓↓↓↓↓↓↓ 逐个调用内部相关的backgroundProcess()方法
    Loader loader = getLoaderInternal();
            loader.backgroundProcess();
    Manager manager = getManagerInternal();
            manager.backgroundProcess();
    Realm realm = getRealmInternal();
            realm.backgroundProcess();
	//调用管道内左右阀的backgroundProcess()方法
    Valve current = pipeline.getFirst();
    while (current != null) {
            current.backgroundProcess();
        current = current.getNext();
    }
    //最后这里注册了一个Lifecycle.PERIODIC_EVENT事件
    fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);
}
```

>  说明一下大概的功能：逐个调用内部相关的backgroundProcess()方法，包括管道内左右阀的backgroundProcess()方法.

### 加载WEB应用事件分析

这里接着上面，默认`ContainerBackgroundProcessor[StandardEngine[Catalina]]`的进程会定期执行各个容器组件的相关的`backgroundProcess `方法，也会定期发布**PERIODIC_EVENT事件**，所有的组件都会收到的。

其中，**Host组件也会收到PERIODIC_EVENT事件**【划重点】来分析这个Host组件对事件的处理

#### 事件处理监听创建

这个处理的方法是在启动解析Host的时候【也就是`org.apache.catalina.startup.Catalina`类中】：

```java
digester.addRuleSet(new HostRuleSet("Server/Service/Engine/")); //这一行代码 为嵌套元素添加规则集
```

这里的HostRuleSet中的规则集addRuleInstances 方法：

```java
public void addRuleInstances(Digester digester) {

    digester.addObjectCreate(prefix + "Host",
                             "org.apache.catalina.core.StandardHost",
                             "className");
    digester.addSetProperties(prefix + "Host");
    digester.addRule(prefix + "Host",
                     new CopyParentClassLoaderRule());
    digester.addRule(prefix + "Host",
                     new LifecycleListenerRule
                     ("org.apache.catalina.startup.HostConfig",
                      "hostConfigClass"));
    digester.addSetNext(prefix + "Host",
                        "addChild",
                        "org.apache.catalina.Container");

    digester.addCallMethod(prefix + "Host/Alias",
                           "addAlias", 0);

    //Cluster configuration start
	.........
}
```

这里可以看到，**在Host的节点中，会添加HostConfig作为StandardHost对象的监听器。**

> 这里响应的事件的处理方法就是在这个HostConfig的处理时间的lifecycleEvent 方法中。

####  分析事件的处理-加载web应用

处理事件的方法是在**HostConfigd的lifecycleRent方法**中：

```java
public void lifecycleEvent(LifecycleEvent event) {
    // Identify the host we are associated with
    try {
        host = (Host) event.getLifecycle();
        if (host instanceof StandardHost) {
            setCopyXML(((StandardHost) host).isCopyXML());
            setDeployXML(((StandardHost) host).isDeployXML());
            setUnpackWARs(((StandardHost) host).isUnpackWARs());
            setContextClass(((StandardHost) host).getContextClass());
        }
    } catch (ClassCastException e) {
       ......
    } 
    // Process the event that has occurred ↓↓↓↓要处理的事件就是这一个↓↓
    if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
        check();
    } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
        beforeStart();
    } else if (event.getType().equals(Lifecycle.START_EVENT)) {
        start();
    } else if (event.getType().equals(Lifecycle.STOP_EVENT)) {
        stop();
    }
}
```

这里处理事件，如果是`PERIODIC_EVENT`事件则`check()`方法，`START_EVENT`事件则`start()`。

而这两个方法的最后都有一个：

```java
 deployApps();
```

这个就是加载web应用的东东了，来看一下具体的实现代码：

```java
/** 为在我们的“应用程序根目录”中找到的任何目录或war文件部署应用程序。
 * Deploy applications for any directories or WAR files that are found
 * in our "application root" directory.
 */
protected void deployApps() {
    File appBase = appBase();
    File configBase = configBase();
    String[] filteredAppPaths = filterAppPaths(appBase.list());
    // Deploy XML descriptors from configBase
    deployDescriptors(configBase, configBase.list());
    // Deploy WARs
    deployWARs(appBase, filteredAppPaths);
    // Deploy expanded folders
    deployDirectories(appBase, filteredAppPaths);
}
```

**这里就是各种方式来发布WEB应用的地方了；【加载WEB应用的地方了】**

> 说明一下，默认情况下组件启动的时候会发布一个`Lifecycle.START_EVENT`事件（在`org.apache.catalina.core.ContainerBase`类的 startInternal 方法倒数第二行），所以默认启动时将会执行 HostConfig 的 start 方法，在该方法的也会调用deployApps（）这个加载应用的方法。
>
> ```java
> if (host.getDeployOnStartup())
>     deployApps();
> ```
>
> 默认配置 host.getDeployOnStartup() 返回 true ，这样容器就会在启动的时候直接加载相应的 web 应用。
>
> 当然，如果在 server.xml 中 Host 节点的 deployOnStartup 属性设置为 false ，则容器启动时不会加载应用，启动完之后不能立即提供 web 应用的服务。但因为有上面提到的后台处理线程在运行，会定期执行 HostConfig 的 check 方法；所以web 应用也会被加载。。

2019--04-30

---

## 二、解析加载web.xml

> 加载web.xml这个就开始一直想了解的Tomcat对WEB应用的加载方式了。

###  获取到war包，启动线程处理

>  最开始的入口位置：HostConfig.lifecycleEvent.deployApps()

在`org.apache.catalina.startup.HostConfig`的 lifecycleEvent 方法中，对Tomcat启动或之后加载web应用进行了实现。就是**deployApps()**这个方法。

```java
//为在我们的“应用程序根目录”中找到的任何目录或war文件部署应用程序。
protected void deployApps() {
    File appBase = appBase();
    File configBase = configBase();
    String[] filteredAppPaths = filterAppPaths(appBase.list());
    // Deploy XML descriptors from configBase   xml文件描述
    deployDescriptors(configBase, configBase.list());
    // Deploy WARs  	WAR包
    deployWARs(appBase, filteredAppPaths);
    // Deploy expanded folders  文件目录
    deployDirectories(appBase, filteredAppPaths);
}
```

> 可以看到这里部署应用有三种方式：XML 文件描述符、WAR 包、文件目录。三种方式部署的总体流程很相似，都是一个 web 应用分配一个线程来处理，这里统一放到与 Host 内部的线程池对象中（ startStopExecutor ），所以有时会看到在默认配置下 Tomcat 启动后可能有一个叫`-startStop-`的线程还会运行一段时间才结束。

这三个的加载方法有很大一部分是一样的，我只想看WAR包加载的，就是**deployWARs**这个方法：

> 方法的代码超级多，都不是啥看得懂的 ，删减到只有重点就行：

```java
protected void deployWARs(File appBase, String[] files) {
    ExecutorService es = host.getStartStopExecutor();
    List<Future<?>> results = new ArrayList<Future<?>>();

    for (int i = 0; i < files.length; i++) {
      ..........
        File war = new File(appBase, files[i]);
        if (files[i].toLowerCase(Locale.ENGLISH).endsWith(".war") &&
                war.isFile() && !invalidWars.contains(files[i]) ) {
			//这里获取到web应用的名称↓↓↓↓↓↓↓
            ContextName cn = new ContextName(files[i], true);
......
    		//下面这个才是要看的，这里使用了ExecutorService线程池进行执行DeployWar类
            results.add(es.submit(new DeployWar(this, cn, war)));
        }
    }
  ......
}
```

然后看一下这个**DeployWar**类的相关代码：

```java
private static class DeployWar implements Runnable {

    private HostConfig config;
    private ContextName cn;
    private File war;

    public DeployWar(HostConfig config, ContextName cn, File war) {
        this.config = config;
        this.cn = cn;
        this.war = war;
    }

    @Override
    public void run() {
        config.deployWAR(cn, war);
    }
}
```

代码相当的简单，这里就是使用线程池执行`config.deployWAR(cn, war);`方法。

> 这个里面就是创建Context的实现的地方了，接下来要分析的东东。

### 创建应用对象Context

在`deployWAR(ContextName cn, File war) `方法中，会有这么一段代码：

```java
host.addChild(context);
```

这里，会创建Context 对象，并且绑定到Host中去了。

然而，到这里，容器还是无法响应浏览器的请求，这需要WEB应用中具体的Servlet来处理的，中间还有相应的过滤器( filter )、监听器( listener )等。

这些配置的信息，是在 web 应用的`WEB-INF\web.xml`文件的。

> servlet3 中已经支持将这些配置信息放到 Java 文件的注解中，但万变不离其宗，总归要在 web 应用的某个地方说明，并在容器启动时加载，这样才能真正提供 web 服务，响应请求

### 构造处理的监听器

> 在上面↑提到的三种部署应用的实现代码中，都有下面的这个共通的代码：

```java
Class<?> clazz = Class.forName(host.getConfigClass());
LifecycleListener listener =
    (LifecycleListener) clazz.newInstance();
context.addLifecycleListener(listener);
........
host.addChild(context);
```

这里的`host.getConfigClass()`获取的就是**StandardHost**类的变量**configClass**，就是：`org.apache.catalina.startup.ContextConfig`。这里会在构建的context中添加新的监听器。

 **开始分析**

**先看StandardHost 的 addChild 方法的：**

```java
public void addChild(Container child) {
    child.addLifecycleListener(new MemoryLeakTrackingListener());
    if (!(child instanceof Context))
        throw new IllegalArgumentException
            (sm.getString("standardHost.notContext"));
    super.addChild(child);  //←←←←←←←←←←←←←看这一个←←←←←←←←←←←←←←←←←
}
```

**父类的addChild**方法：

```java
public void addChild(Container child) {
    if (Globals.IS_SECURITY_ENABLED) {
        PrivilegedAction<Void> dp =
            new PrivilegedAddChild(child);
        AccessController.doPrivileged(dp);
    } else {
        addChildInternal(child);//←←←←←←←←←←←看这里←←←←←←←←←←←←←←←←←←←
    }
}
```

**addChildInternal(child)** 方法：

```java
private void addChildInternal(Container child) {

    if( log.isDebugEnabled() )
        log.debug("Add child " + child + " " + this);
    synchronized(children) {
        if (children.get(child.getName()) != null)
            throw new IllegalArgumentException("addChild:  Child name '" +
                                               child.getName() +
                                               "' is not unique");
        child.setParent(this);  // May throw IAE
        children.put(child.getName(), child);
    }
    try {
        if ((getState().isAvailable() ||
                LifecycleState.STARTING_PREP.equals(getState())) &&
                startChildren) {
            child.start();   //←←←←←←←←←看这里←←←←←←←←←←←
        }
    } catch (LifecycleException e) {	.......
    } finally {
        fireContainerEvent(ADD_CHILD_EVENT, child);
    }
}
```

最终会调用子容器start 方法，这里就是StandardContext 的 start 方法。

即给 host 对象添加子容器时将会调用子容器的 start 方法，调用 StandardContext 的 start 方法最终会调用`org.apache.catalina.core.StandardContext`类的 startInternal 方法。并且发布一系列的事件：包括：

`BEFORE_INIT_EVENT`、`AFTER_INIT_EVENT`、`BEFORE_START_EVENT`、`CONFIGURE_START_EVENT`、`START_EVENT`、`AFTER_START_EVENT`

而在上面的代码（执行`host.addChild(context);`之前），在Context中注册了一个监听器：`org.apache.catalina.startup.ContextConfig`。看一下这个监听器对start发布的一系列事件的处理。

### 监听器对加载事件的处理

直接来看一下监听器的处理方法，也就是`org.apache.catalina.startup.ContextConfig.lifecycleEvent`:

```java
public void lifecycleEvent(LifecycleEvent event) {
    try {
        context = (Context) event.getLifecycle();
    } catch (ClassCastException e) ......   }
    if (event.getType().equals(Lifecycle.CONFIGURE_START_EVENT)) {
        configureStart(); //←←←←←←第三个处理事件←←←←←←←←←
    } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
        beforeStart(); //←←←←←←第二个处理事件←←←←←←←←←
    } else if (event.getType().equals(Lifecycle.AFTER_START_EVENT)) {
        // Restore docBase for management tools
        if (originalDocBase != null) {
            context.setDocBase(originalDocBase);
        }
    } else if (event.getType().equals(Lifecycle.CONFIGURE_STOP_EVENT)) {
        configureStop();
    } else if (event.getType().equals(Lifecycle.AFTER_INIT_EVENT)) {
        init();   //←←←←←←第一个处理事件←←←←←←←←←
    } else if (event.getType().equals(Lifecycle.AFTER_DESTROY_EVENT)) {
        destroy();
    }
}
```

这里按照发布时间的顺序，会响应调用对应的处理方法，如上面标注的顺序。

这里来看一下第三个处理事件，`configureStart()`方法：

```java
public class ContextConfig implements LifecycleListener {
protected synchronized void configureStart() {
    // Called from StandardContext.start()
    ......
    webConfig();  //这个方法就是解析web.xml的实现了 *****看这里*****

    if (!context.getIgnoreAnnotations()) {
        applicationAnnotationsConfig();
    }
    if (ok) {
        validateSecurityRoles();
    }

    // Configure an authenticator if we need one
    if (ok)
        authenticatorConfig();

    // Dump the contents of this pipeline if requested 如果请求，则转储此管道的内容
    if ((log.isDebugEnabled()) && (context instanceof ContainerBase)) {
        log.debug("Pipeline Configuration:");
        Pipeline pipeline = ((ContainerBase) context).getPipeline();
        Valve valves[] = null;
        if (pipeline != null)
            valves = pipeline.getValves();
        if (valves != null) {
            for (int i = 0; i < valves.length; i++) {
                log.debug("  " + valves[i].getInfo());
            }
        }
    }
    // Make our application available if no problems were encountered
  .......

}
```

### 解析web.xml

这里的处理方法中，直接就调动了**webConfig()**,这个方法是具体的处理web.xml的实现方法：

```java
/**
*扫描应用于Web应用程序的web.xml文件并合并它们
*使用全局web.xml文件规范中定义的规则，
*如果存在重复配置，则最特定的级别将获胜。工业工程
*应用程序的web.xml优先于主机级别或全局级别
* Web.xml文件。
*/
protected void webConfig() {

    Set<WebXml> defaults = new HashSet<WebXml>();
    defaults.add(getDefaultWebXmlFragment());

    WebXml webXml = createWebXml();

    // Parse context level web.xml
    InputSource contextWebXml = getContextWebXmlSource();
    parseWebXml(contextWebXml, webXml, false);

    ServletContext sContext = context.getServletContext();

    // Ordering is important here

    // Step 1. Identify all the JARs packaged with the application
    // If the JARs have a web-fragment.xml it will be parsed at this
    // point.
    Map<String,WebXml> fragments = processJarsForWebFragments(webXml);

    // Step 2. Order the fragments. 整理碎片
    Set<WebXml> orderedFragments = null;
    orderedFragments =
            WebXml.orderWebFragments(webXml, fragments, sContext);

    // Step 3. Look for ServletContainerInitializer implementations 
    if (ok) {   //看servletcontainerinitializer的实现等
        processServletContainerInitializers();
    }

    if  (!webXml.isMetadataComplete() || typeInitializerMap.size() > 0) {
        // Step 4. Process /WEB-INF/classes for annotations 注释的进程/WEB-INF/类
        if (ok) {
            // Hack required by Eclipse's "serve modules without
            // publishing" feature since this backs WEB-INF/classes by
            // multiple locations rather than one.
            NamingEnumeration<Binding> listBindings = null;
            try {
                try {
                    listBindings = context.getResources().listBindings(
                            "/WEB-INF/classes");
                } catch (NameNotFoundException ignore) {
                    // Safe to ignore
                }
                while (listBindings != null &&
                        listBindings.hasMoreElements()) {
                    Binding binding = listBindings.nextElement();
                    if (binding.getObject() instanceof FileDirContext) {
                        File webInfClassDir = new File(
                                ((FileDirContext) binding.getObject()).getDocBase());
                        processAnnotationsFile(webInfClassDir, webXml,
                                webXml.isMetadataComplete());
                    } else if ("META-INF".equals(binding.getName())) {
                        // Skip the META-INF directory from any JARs that have been
                        // expanded in to WEB-INF/classes (sometimes IDEs do this).
                    } else {
                        String resource =
                                "/WEB-INF/classes/" + binding.getName();
                        try {
                            URL url = sContext.getResource(resource);
                            processAnnotationsUrl(url, webXml,
                                    webXml.isMetadataComplete());
                        } catch (MalformedURLException e) {
                            log.error(sm.getString(
                                    "contextConfig.webinfClassesUrl",
                                    resource), e);
                        }
                    }
                }
            } catch (NamingException e) {
                log.error(sm.getString(
                        "contextConfig.webinfClassesUrl",
                        "/WEB-INF/classes"), e);
            }
        }

        // Step 5. Process JARs for annotations - only need to process 为注释处理JAR-只需要处理
        // those fragments we are going to use
        if (ok) {
            processAnnotations(
                    orderedFragments, webXml.isMetadataComplete());
        }

        // Cache, if used, is no longer required so clear it
        javaClassCache.clear();
    }

    if (!webXml.isMetadataComplete()) {
        // Step 6. Merge web-fragment.xml files into the main web.xml
        // file. 将web-fragment.xml文件合并到主web.xml中
        if (ok) {
            ok = webXml.merge(orderedFragments);
        }

        // Step 7. Apply global defaults 应用全局默认值
        // Have to merge defaults before JSP conversion since defaults  
        //在JSP转换之前必须合并默认值，因为默认值
        // provide JSP servlet definition.提供JSP servlet定义
        webXml.merge(defaults);

        // Step 8. Convert explicitly mentioned jsps to servlets 将显式提到的JSP转换为servlet
        if (ok) {
            convertJsps(webXml);
        }

        // Step 9. Apply merged web.xml to Context
        if (ok) {
            webXml.configureContext(context);  //++++++看下这里+++++
        }
    } else {
        webXml.merge(defaults);
        convertJsps(webXml);
        webXml.configureContext(context);   //++++++看下这里+++++
    }

    // Step 9a. Make the merged web.xml available to other
    // components, specifically Jasper, to save those components
    // from having to re-generate it.
    // TODO Use a ServletContainerInitializer for Jasper
    String mergedWebXml = webXml.toXml();
    sContext.setAttribute(
           org.apache.tomcat.util.scan.Constants.MERGED_WEB_XML,
           mergedWebXml);
    if (context.getLogEffectiveWebXml()) {
        log.info("web.xml:\n" + mergedWebXml);
    }

    // Always need to look for static resources
    // Step 10. Look for static resources packaged in JARs
    if (ok) {
        // Spec does not define an order.
        // Use ordered JARs followed by remaining JARs
        Set<WebXml> resourceJars = new LinkedHashSet<WebXml>();
        for (WebXml fragment : orderedFragments) {
            resourceJars.add(fragment);
        }
        for (WebXml fragment : fragments.values()) {
            if (!resourceJars.contains(fragment)) {
                resourceJars.add(fragment);
            }
        }
        processResourceJARs(resourceJars);
        // See also StandardContext.resourcesStart() for
        // WEB-INF/classes/META-INF/resources configuration
    }

    // Step 11. Apply the ServletContainerInitializer config to the
    // context
    if (ok) {
        for (Map.Entry<ServletContainerInitializer,
                Set<Class<?>>> entry :
                    initializerClassMap.entrySet()) {
            if (entry.getValue().isEmpty()) {
                context.addServletContainerInitializer(
                        entry.getKey(), null);
            } else {
                context.addServletContainerInitializer(
                        entry.getKey(), entry.getValue());
            }
        }
    }
}
```

对以上代码的总结：

**概括起来包括合并 Tomcat 全局 web.xml 、当前应用中的 web.xml 、web-fragment.xml 和 web 应用的注解中的配置信息，并将解析出的各种配置信息（如 servlet 配置、filter 配置等）关联到 Context 对象中。**

>配置信息取到之后对配置信息中的对象进行实例化，我找到的地方是在这一段代码：
>
>```java
>processAnnotationsFile(webInfClassDir, webXml,
>                           webXml.isMetadataComplete());
>```
>
>然后调用：`processAnnotationsStream(fis, fragment, handlesTypesOnly);`;
>
>**processAnnotationsStream**的方法实现：
>
>```java
>protected void processAnnotationsStream(InputStream is, WebXml fragment,
>       boolean handlesTypesOnly)
>       throws ClassFormatException, IOException {
>
>   ClassParser parser = new ClassParser(is);
>   JavaClass clazz = parser.parse();
>   checkHandlesTypes(clazz);
>
>   if (handlesTypesOnly) {
>       return;
>   }
>
>   String className = clazz.getClassName();
>
>   AnnotationEntry[] annotationsEntries = clazz.getAnnotationEntries();
>   if (annotationsEntries != null) {
>       for (AnnotationEntry ae : annotationsEntries) {
>           String type = ae.getAnnotationType();
>           if ("Ljavax/servlet/annotation/WebServlet;".equals(type)) {
>               processAnnotationWebServlet(className, ae, fragment);
>           }else if ("Ljavax/servlet/annotation/WebFilter;".equals(type)) {
>               processAnnotationWebFilter(className, ae, fragment);
>           }else if ("Ljavax/servlet/annotation/WebListener;".equals(type)) {
>               fragment.addListener(className);
>           } else {
>               // Unknown annotation - ignore
>           }
>       }
>   }
>}
>```
>
>这里最后的三个方法`processAnnotationWebServlet`,`processAnnotationWebFilter`,`fragment.addListener`，会创建的实例对象添加到WebXml对象的成员变量中。
>
>```java
>//在processAnnotationWebServlet方法中：
>fragment.addServlet(servletDef);
>//在processAnnotationWebFilter方法中：
>fragment.addFilter(filterDef);
>```
>
>对，这里只是把配置的Filter，Listener，Servlet 的配置信息保存下来以便之后关联到Context中去【这里只是封装类，并没有构造出web 应用中的 Servlet、Listener、Filter 的实例】
>
>> 这个在后面一个章节会有具体的说明。

### web配置信息实例化关联Context

这里特别说明一下，获取到所有的配置信息时候，**关联到Context对象的处理方法**；

代码为114行的，**以及具体的实现方法**：

```java
webXml.configureContext(context);
```

超级长的代码【所以都删了 反正不会认真看╮(╯_╰)╭】:

```java
/**
 * Configure a {@link Context} using the stored web.xml representation.
 * 使用存储的web.xml表示形式配置@link上下文。
 * @param context   The context to be configured
 * @param 上下文要配置的上下文*/
public void configureContext(Context context) {
    // As far as possible, process in alphabetical order so it is easy to
    // check everything is present
    // Some validation depends on correct public ID
    context.setPublicId(publicId);

    // Everything else in order
    context.setEffectiveMajorVersion(getMajorVersion());
    context.setEffectiveMinorVersion(getMinorVersion());

    for (Entry<String, String> entry : contextParams.entrySet()) {
        context.addParameter(entry.getKey(), entry.getValue());
    }
    context.setDisplayName(displayName);
    context.setDistributable(distributable);
    for (ContextLocalEjb ejbLocalRef : ejbLocalRefs.values()) {
        context.getNamingResources().addLocalEjb(ejbLocalRef);
    }
    for (ContextEjb ejbRef : ejbRefs.values()) {
        context.getNamingResources().addEjb(ejbRef);
    }
    for (ContextEnvironment environment : envEntries.values()) {
        context.getNamingResources().addEnvironment(environment);
    }
    for (ErrorPage errorPage : errorPages.values()) {
        context.addErrorPage(errorPage);
    }
    for (FilterDef filter : filters.values()) {
        if (filter.getAsyncSupported() == null) {
            filter.setAsyncSupported("false");
        }
        context.addFilterDef(filter);
    }
   ......
```

这里就没什么好看的了，就是用了各种 set、add 方法，从而将 web.xml 中的各种配置信息与表示一个 web 应用的 context 对象关联起来。

> Context 就是一个WEB应用，里面的东东就是web应用的各种东东了。
>
> 注意：这里只是关联上，并没有具体的实现的实例化哦。。。。



---

2019-05-08 小杭   ε=(´ο｀*)))唉   学习好累





