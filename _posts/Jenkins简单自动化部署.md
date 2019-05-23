# Jenkins 测试部署学习

> Jenkins + maven + SVN + shell 测试自动部署（好像并不自动，要手动点一下）

[TOC]

---

## 简介说明一下

Jenkins只是个平台，真正运作的是它的插件，主要是啥功能的插件它都有。

最常用来使用的目的是：作为持续集成工具。

> 持续集成是一种软件开发实践，即团队开发成员经常集成他们的工作，通常每个成员至少集成一次，也就意味着每天可能会发生多次集成。每次集成都通过自动化的构建（包括编译，发布，自动化测试）来验证，从而尽快地发现集成错误。许多团队发现这个过程可以大大减少集成的问题，让团队能够更快的开发内聚的软件。
>
> 我感觉上就是个自动编译打包部署的东东。

## 下载安装Jenkins

### 一些资源地址

* 插件中心：https://plugins.jenkins.io/
* 官网：<https://jenkins.io/>

### 安装

下载个war的包，然后使用Tomcat，或者java -jar 启动就可以了。非常的简单。

我用的是：

```shell
nohup java  -Dhudson.util.ProcessTree.disable=true -jar jenkins.war --httpPort=8765  > /dev/null 2>&1 &
```

到这里就可以访问了：http://192.168.1.212:8765/。【这里的配置在后面会说到】

第一次登入的时候会要求配置一些相关的信息，这里如果可以联网，**一定要原则默认安装必要的插件，**不然像我一样手动安装真的很费时间。。。(ノ｀Д)ノ

> 这里注意下，Jenkins 的配置等信息会默认在系统`/root/.jenkins/`这个目录下面，包括之后新建的任务，工作空间等等。。



## 手动离线安装插件

Jenkins系统里面，最重要的就是插件了。系统本身在`系统管理 》 插件管理`里面就可以在线管理了（安装，更新等）。

然而，我这次不行，提示是连不上。（这个很容易解决的）

### 这里先说一下手动离线安装

插件hpi可以去插件中心下载，这里我安装的插件有：

* maven
* svn
* pubblish-oer-ssh 【这个是为了执行shell脚本不被杀点进程弄的，也不知道有没有效果】

安装方式是在 **插件管理**的页面的  **Advanced** 标签页面下，**Upload Plugin**处上传插件hpi文件就可以了。

**这里需要注意的是，安装这些大的插件，本身会有很多依赖插件，所以要按依赖顺序安装。**

> 就是在这个地方花了很多时间，依赖会有很多，及其麻烦。。。ε=(´ο｀*)))唉

### 在线管理-安装更新

我没有用这种方式安装插件，简直就是浪费时间了。

**用这种方式，就搜索需要插件，然后勾选上，点一下，去喝茶等着就可以了。**

#### 碰到的问题

提示：`There were errors checking the update sites: SSLHandshakeException: sun.security.validator.Validator`这的报错，导致无法chack 插件信息。

**解决：**

 **Advanced** 标签页面下，**Update Site**选项的URL：配置为 http://updates.jenkins.io/update-center.json  即可！！！



## 创建 maven + svn + shell 的自动部署任务

### 前提 

在**系统管理 -> 全局工具配置** 中先配置好JDK和Maven的路径。没有的需要先安装。

### 配置

* 创建新任务 。。。略过

* General 配置 选择丢弃旧的构建 【据说里面的保留数量要填，我是测试环境使用就都不管了╮(╯_╰)╭】

* **Source Code Management**

  * 这里选择**Subversion**
  * 填写URL，和**Credentials**用户密码
  * Repository depth ：infinity
  * 勾选：Ignore externals 和 Cancel process on externals fail 和 Quiet check-out
  * **Check-out Strategy :** Use 'svn update' as much as possible
  * Repository browser ： Auto

* **Build Triggers** 构建模式：Build whenever a SNAPSHOT dependency is built

* **Build**

  * Root POM ：pom.xml
  * Goals and options : clean install

* **Post Steps** 构建完成后的操作

  * 选择：Run only if build succeeds

  * command: 为执行的shell 脚本

  * > rm -f /u01/csb/sys/admin-console-1.1.jar
    > cp /root/.jenkins/workspace/创收宝测试版/admin-console/target/admin-console-1.1.jar /u01/csb/sys/
    >
    > cd /u01/csb/sys/
    >
    > BUILD_ID=dontKillMe ./service.sh start admin 
    >
    >
    > echo "Execute shell Finish"

**OK ,这样配置好就可以了！点个下构建，就可以喝茶等着部署好了。**

### 遇到问题

问题：构建完成后调用shell 脚本里面，再调用系统中的 启动脚本 ，结果没有执行完成。

原因：jenkins默认在build结束后会kill掉所有的衍生进程

解决：

* 使用java -jar启动，-Dhudson.util.ProcessTree.disable=true -jar jenkins.war
* 在execute shell输入框中加入BUILD_ID=DONTKILLME

我这里是上面两个解决方式都用上了。

> 并且，这里的执行shell 脚本，最好是要cd到目录下`./shell`来执行，不然我测试的感觉都不行╮(╯_╰)╭



---

##  完成！

Jenkins 简单的部署，就这个样子了，其实很简单的，点点配置就可以了ε=(´ο｀*)))唉

当然，Jenkins 上还有一堆的插件，一堆的功能，以后有发现在学习了。。。。



2019-03-23 小杭

----

