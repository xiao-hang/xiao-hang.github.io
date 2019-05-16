---
layout: post
#标题配置
title:   Tomcat 源码分析（三）-WEB加载原理（三）-自动加载类及检测文件变动原理
#时间配置
date:   2019-05-16 19:00:00 +0800
#大类配置
categories: 源码
#小类配置
tag: Tomcat
---

* content
{:toc}



## Tomcat 7 自动加载类及检测文件变动原理

### 关于开发工具中的自动加载

在常用的web应用开发工具（如 Eclipse、IntelJ ）中都有集成Tomcat，这样可以将开发的web项目直接发布到tomcat中去。这里会遇到一种情况，在修改了一个文件后，开发工具可以直接编译class文件发布到tomcat的web工程里面。如果tomcat没有配置自动加载功能，JVM中的还是就的class，就需要手动进行restart。

所以，这里说一下**tomcat提供的配置自动加载的配置属性：**

```java
`<Context path="/HelloWorld" docBase="C:/apps/apache-tomcat/DeployedApps/HelloWorld" reloadable="true"/>`
```

**就是`reloadable="true"`这个属性**，这样 Tomcat 就会监控所配置的 web 应用实际路径下的`/WEB-INF/classes`和`/WEB-INF/lib`两个目录下文件的变动，如果发生变更 tomcat 将会自动重启该应用。

### 分析Tomcat自动加载的实现

自动加载的实现，先从Tomcat在启动之后会有一个后台线程，

```java
ContainerBackgroundProcessor[StandardEngine[Catalina]]
```

定时【默认10秒】执行Engine、Host、Context、Wrapper 各容器组件及与它们相关的其它组件的 backgroundProcess 方法。- 这里开始分析。

这个方法被定义在，所有容器组件的父类`org.apache.catalina.core.ContainerBase`类的 backgroundProcess`方法中：

```java
/**
*执行定期任务，如重新加载等。此方法将
*在该容器的类加载上下文中调用。意外的
*丢弃物将被捕获并记录。
 */
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
    //**********现在要看看的是上面↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑这个的地方了*******
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
    //最后这里注册了一个Lifecycle.PERIODIC_EVENT事件  之前分析加载web是在这个事件的处理中
    fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);
}
```

这里与自动加载的代码是Loader :` Loader loader = getLoaderInternal();`,`loader.backgroundProcess();`这两段。

> 这里看一下这个loader 变量是什么时候初始化的：【在StandardContext的startInternal 方法中】
>
> ```java
> if (getLoader() == null) {
>     WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
>     webappLoader.setDelegate(getDelegate());
>     setLoader(webappLoader);
> }
> ```
>
> 这里可以看到这里这个设置的Loader 的类是WebappLoader。

然后具体的关联是在Loader的backgroundProcess()中：

```java
//public class WebappLoader extends LifecycleMBeanBase implements Loader, PropertyChangeListener {
@Override
public void backgroundProcess() {
    if (reloadable && modified()) {
        try {
            Thread.currentThread().setContextClassLoader
                (WebappLoader.class.getClassLoader());
            if (container instanceof StandardContext) {
                ((StandardContext) container).reload();
            }
        } finally {......}
    } else {
        closeJARs(false);
    }
}
```

这里可以看到，这里的条件是reloadable和modified()，这里的reloadable就是配置Context节点的reloadable属性值，而modified()这个方法是对检查文件变动的，之后会分析。

先来看一下，最终要执行的**重新加载的方法：StandardContext类的reload():**

```java
public synchronized void reload() {
......
    // Stop accepting requests temporarily.
    setPaused(true);
    try {
        stop();
    } catch (LifecycleException e) {.......}
    try {
        start();
    } catch (LifecycleException e) {....... }
    setPaused(false);
......
}
```

这里的reload方法中，将执行stop方法将原有的该 web 应用停掉，再调用 start 方法启动该 Context 。

start方法，则会重新加载启动web应用。【就像之前分析的那样_(:з」∠)_】

### 检测文件变动分析

前面，进行reload重新启动web应用的条件为：`if (reloadable && modified()) {`,一个为配置值，另一个就是接下来要说的了。- **modified()**

```java
//public class WebappLoader extends LifecycleMBeanBase  implements Loader, PropertyChangeListener {
public boolean modified() {
    return classLoader != null ? classLoader.modified() : false ;
}
```

这里进行判断的的实际方法是：**WebappLoader 的实例变量 classLoader 的 modified 方法。**

> 说明个Tomcat中加载器的东东，每个web应用会对一个Context节点，在JVM中就会对应一个`org.apache.catalina.core.StandardContext`对象，而每一个StandardContext对象内部都一个加载器实例loader实例变量。可以看到前面说明，这个loader实际上是WebappLoader对象。
>
> 而每一个 WebappLoader 对象内部关联了一个 classLoader 变量（就这这个类的定义中，可以看到该变量的类型是`org.apache.catalina.loader.WebappClassLoader`）。
>
> 所以，这里一个web应用会对应一个StandardContext 一个WebappLoader 一个WebappClassLoader 。

#### WebappLoader 的初始化

WebappLoader的初始化在StandardContext 的初始化的时候已经完成了。上文中已有了：

```java
if (getLoader() == null) {
 WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
 webappLoader.setDelegate(getDelegate());
 setLoader(webappLoader);
}
......
 if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();
```

这里要的代码是先初始化了，之后执行了loader的start()方法，因为WebappLoader 本身也是继承了LifecycleBase 类，所以这里的start()方法，最终也会执行到类自定义的startInternal 方法。

**WebappLoader.startInternal ()方法的源码：**

```java
//public class WebappLoader extends LifecycleMBeanBase implements Loader, PropertyChangeListener {
@Override
protected void startInternal() throws LifecycleException {
  ......
	// 为JNDI协议注册流处理程序工厂 ?? 啥意思啊 ╮(╯_╰)╭
    // Register a stream handler factory for the JNDI protocol
    URLStreamHandlerFactory streamHandlerFactory =
            DirContextURLStreamHandlerFactory.getInstance();
  	......
            URL.setURLStreamHandlerFactory(streamHandlerFactory);
	......
    }
	// ********基于当前存储库列表构造类加载器*********需要看的就是这一段
    // Construct a class loader based on our current repositories list
    try {
        classLoader = createClassLoader();   // 开始就调用了这个创建加载器的方法
        classLoader.setJarOpenInterval(this.jarOpenInterval);
        classLoader.setResources(container.getResources());
        classLoader.setDelegate(this.delegate);
        classLoader.setSearchExternalFirst(searchExternalFirst);
        if (container instanceof StandardContext) {
            classLoader.setAntiJARLocking(
                    ((StandardContext) container).getAntiJARLocking());
            classLoader.setClearReferencesRmiTargets(
                    ((StandardContext) container).getClearReferencesRmiTargets());
            classLoader.setClearReferencesStatic(
                    ((StandardContext) container).getClearReferencesStatic());
            classLoader.setClearReferencesStopThreads(
                    ((StandardContext) container).getClearReferencesStopThreads());
            classLoader.setClearReferencesStopTimerThreads(
                    ((StandardContext) container).getClearReferencesStopTimerThreads());
            classLoader.setClearReferencesHttpClientKeepAliveThread(
                    ((StandardContext) container).getClearReferencesHttpClientKeepAliveThread());
            classLoader.setClearReferencesObjectStreamClassCaches(
                    ((StandardContext) container).getClearReferencesObjectStreamClassCaches());
        }
        for (int i = 0; i < repositories.length; i++) {
            classLoader.addRepository(repositories[i]);
        }
        // Configure our repositories
        setRepositories();
        setClassPath();
        setPermissions();
        ((Lifecycle) classLoader).start();
        // Binding the Webapp class loader to the directory context
       .....
    } catch (Throwable t) {......}
    setState(LifecycleState.STARTING);
}
    /**
     * Create associated classLoader. 创建关联的类加载器。
     * 这里反射实例化了一个WebappClassLoader 对象。
     */
    private WebappClassLoaderBase createClassLoader()
        throws Exception {

        Class<?> clazz = Class.forName(loaderClass);  
        WebappClassLoaderBase classLoader = null;

        if (parentClassLoader == null) {
            parentClassLoader = container.getParentClassLoader();
        }
        Class<?>[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        Constructor<?> constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoaderBase) constr.newInstance(args);

        return classLoader;

    }
```

这里，就分析了这个要使用的类的初始化过程了。

#### WebappClassLoader 的 modified 方法-检测变动的代码

可以再前边看到，判断文件变动的检测代码为modified()方法：

```java
classLoader != null ? classLoader.modified() : false ；
```

就是这句代码，所以来看一下这个**classLoader.modified()**也就是WebappClassLoader 的：

```java
public boolean modified() {
  ......s
    // Checking for modified loaded resources
    int length = paths.length;
    int length2 = lastModifiedDates.length;
    if (length > length2)
        length = length2;
//****这里对比资源文件里面的文件的最后修改时间是否一致，以便判断是否变动****
    for (int i = 0; i < length; i++) {
        try {
            long lastModified =
                ((ResourceAttributes) resources.getAttributes(paths[i]))
                .getLastModified();
            if (lastModified != lastModifiedDates[i]) {
              ......
                return (true);
            }
        } catch (NamingException e) {......return (true);}
    }

    length = jarNames.length;

    // Check if JARs have been added or removed
    if (getJarPath() != null) {
        try {
            NamingEnumeration<Binding> enumeration =
                resources.listBindings(getJarPath());
            int i = 0;
            while (enumeration.hasMoreElements() && (i < length)) {
                NameClassPair ncPair = enumeration.nextElement();
                String name = ncPair.getName();
                // Ignore non JARs present in the lib folder
                if (!name.endsWith(".jar"))
                    continue;
                if (!name.equals(jarNames[i])) {
                    // Missing JAR
                 ......
                    return (true);
                }
                i++;
            }
            if (enumeration.hasMoreElements()) {
                while (enumeration.hasMoreElements()) {
                    NameClassPair ncPair = enumeration.nextElement();
                    String name = ncPair.getName();
                    // Additional non-JAR files are allowed
                    if (name.endsWith(".jar")) {
                        // There was more JARs
                        log.info("    Additional JARs have been added");
                        return (true);
                    }
                }
            } else if (i < jarNames.length) {
                // There was less JARs
                log.info("    Additional JARs have been added");
                return (true);
            }
        } catch (NamingException e) {.......}
    }
    // No classes have been modified
    return (false);
}
```

这段代码从总体上看共分成两部分，第一部分检查 web 应用中的 class 文件是否有变动，根据 class 文件的最近修改时间来比较，如果有不同则直接返回`true`，如果 class 文件被删除也返回`true`。

第二部分检查 web 应用中的 jar 文件是否有变动，如果有同样返回`true`。

> 这里的代码看起来，还是比较容易理解的╮(╯_╰)╭

#### 关于当前资源信息获取

关于，检查文件变动的关键代码就是：

```java
 long lastModified =
                ((ResourceAttributes) resources.getAttributes(paths[i]))
                .getLastModified();
            if (lastModified != lastModifiedDates[i]) {
```

WebappClassLoader 的实例变量`resources`中取出文件当前的最近修改时间，与 WebappClassLoader 原来缓存的该文件的最近修改时间做比较。

**这里看一下 resources.getAttributes 方法：**

这里的resources实际上的是`javax.naming.directory.DirContext`类，看下初始化的地方，在WebappLoader 的 startInternal 方法中：【就在上面的】

```java
 classLoader.setResources(container.getResources()); //这里设置的，是在StandardContext初始化的时候
 ((Lifecycle) classLoader).start();
```

**StandardContext 中 resources 是怎么赋值：**StandardContext 的 startInternal 方法中

```java
// Add missing components as necessary
if (webappResources == null) {   // (1) Required by Loader
    try {
        if ((getDocBase() != null) && (getDocBase().endsWith(".war")) &&
                (!(new File(getBasePath())).isDirectory()))
            setResources(new WARDirContext()); //我们常用的wer发布加载的是这个
        else
            setResources(new FileDirContext()); //默认的应用是文件发布的
    } catch (IllegalArgumentException e) {......ok = false;}
}
if (ok) {
    if (!resourcesStart()) {...... } //在这里做了初始化
}
```

这里会对resources进行赋值，并且初始化；**看下resourcesStart()初始化的方法:**

```java
//public class StandardContext extends ContainerBase
public boolean resourcesStart() {
......
    try {
        ProxyDirContext proxyDirContext =
            new ProxyDirContext(env, webappResources);
       ......// 中间的太多不知道啥的东西 (ノ｀Д)ノ 
        super.setResources(proxyDirContext);   //要看的就只是这个
    } catch (Throwable t) {......}
    return (ok);
}
```

**很明显，这里的resources 赋的是 proxyDirContext 对象**，而 proxyDirContext 是一个代理对象，代理的就是 webappResources ，按上面的描述即`org.apache.naming.resources.FileDirContext`。

> `org.apache.naming.resources.FileDirContext`继承自抽象父类`org.apache.naming.resources.BaseDirContext`，而 BaseDirContext 又实现了`javax.naming.directory.DirContext`接口。所以 JNDI 操作中的 lookup、bind、getAttributes、rebind、search 等方法都已经在这两个类中实现了。当然里面还有 JNDI 规范之外的方法如 list 等。

所以，接下来看看一下这个**getAttributes 方法的调用**。

> 最终都会调用到抽象方法 doGetAttributes 的。
>
> ```java
> //public abstract class BaseDirContext implements DirContext {
> public final Attributes getAttributes(String name, String[] attrIds)
>     throws NamingException {
> ......
>     // Next do a standard lookup
>     Attributes attrs = doGetAttributes(name, attrIds);
> ```

**看一下FileDirContext 的doGetAttributes定义：**

```java
protected Attributes doGetAttributes(String name, String[] attrIds)
    throws NamingException {
    // Building attribute list
    File file = file(name, true);
    if (file == null)         return null;
    return new FileResourceAttributes(file);
}
```

到这里就可以了，最终是调用了File的东西【java文件操作】。

实际就是根据传入的文件名查找目录下是否存在该文件，如果存在则返回包装了的文件属性对象 FileResourceAttributes 。 FileResourceAttributes 类实际是对`java.io.File`类做了一层包装。

#### 关于已加载类的资源信息

还有两个内置变量`paths`和`lastModifiedDates`值究竟什么时候赋的呢？

> 说一下 WebappClassLoader 这个自定义类加载器的用法，在 Tomcat 中所有 web 应用内`WEB-INF\classes`目录下的 class 文件都是用这个类加载器来加载的，一般的自定义加载器都是覆写 ClassLoader 的 findClass 方法，这里也不例外。WebappClassLoader 覆盖的是 URLClassLoader 类的 findClass 方法，而在这个方法内部最终会调用`findResourceInternal(String name, String path)`方法：
>
> ```java
> // Register the full path for modification checking
> // Note: Only syncing on a 'constant' object is needed
> synchronized (allPermission) {
>     int j;
>     long[] result2 =
>         new long[lastModifiedDates.length + 1];
>     for (j = 0; j < lastModifiedDates.length; j++) {
>         result2[j] = lastModifiedDates[j];
>     }
>     result2[lastModifiedDates.length] = entry.lastModified;
>     lastModifiedDates = result2;
> 
>     String[] result = new String[paths.length + 1];
>     for (j = 0; j < paths.length; j++) {
>         result[j] = paths[j];
>     }
>     result[paths.length] = fullPath;
>     paths = result;
> 
> }
> ```
>
> 这里可以看到在**加载一个新的 class 文件时会给 WebappClassLoader 的实例变量`lastModifiedDates`和`paths`数组添加元素。**这里就解答了上面提到的文件变更比较代码的疑问。要说明的是在 tomcat 启动后 web 应用中所有的 class 文件并不是全部加载的，而是配置在 web.xml 中描述的需要与应用一起加载的才会立即加载，否则只有到该类首次使用时才会由类加载器加载。

而关于 jar 包文件变动的比较代码同 class 文件比较的类似，同样是取出当前 web 应用`WEB-INF\lib`目录下的所有 jar 文件，与 WebappClassLoader 内部缓存的`jarNames`数组做比较，如果文件名不同或新加或删除了 jar 文件都返回`true`。

> 这里 jarNames 变量的初始赋值代码在 WebappClassLoader 类的 addJar 方法中的开头部分........
>
> 最后这一点点，看不下去了    (╯‵□′)╯︵┻━┻

### 结束

2019-05-16 小杭

---

## 参考资料

* ### 源码分析三：web 应用加载原理

  * [《Tomcat 7 中 web 应用加载原理（一）Context 构建》](http://www.iocoder.cn/Tomcat/yuliu/Web-application-loading-principle-1-Context-construction)
  * [《Tomcat 7 中 web 应用加载原理（二）web.xml 解析》](http://www.iocoder.cn/Tomcat/yuliu/Web-application-loading-principle-2-web-xml-parsing)
  * [《Tomcat 7 中 web 应用加载原理（三）Listener、Filter、Servlet 的加载和调用》](http://www.iocoder.cn/Tomcat/yuliu/Web-application-loading-principle-3-Listener-Filter-Servlet)
  * [《Tomcat 7 自动加载类及检测文件变动原理》](http://www.iocoder.cn/Tomcat/yuliu/Automatic-loading-of-classes-and-detection-of-file-changes)

---

