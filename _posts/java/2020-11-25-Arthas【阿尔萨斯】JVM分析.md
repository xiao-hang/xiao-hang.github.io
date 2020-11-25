---
layout: post
#标题配置
title:   Arthas【阿尔萨斯】JVM分析
#时间配置
date:   2020-11-25 12:00:00 +0800
#大类配置
categories: 服务组件
#小类配置
tag: Arthas
---

* content
{:toc}




> Arthas（阿尔萨斯）是阿里巴巴开源的 Java 诊断工具，深受开发者喜爱。
>
> 当你遇到以下类似问题而束手无策时，Arthas 可以帮助你解决：
>
> 1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
> 2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
> 3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
> 4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
> 5. 是否有一个全局视角来查看系统的运行状况？
> 6. 有什么办法可以监控到JVM的实时运行状态？
>
> Arthas 采用命令行交互模式，同时提供丰富的 `Tab` 自动补全功能，进一步方便进行问题的定位和诊断。

### 计划

Arthas 【阿尔萨斯】

* Alibaba开源的Java诊断工具。
* 使用教程：https://www.jianshu.com/p/507f7e0cc3a3
* 官网：https://arthas.aliyun.com/doc/   【官方的文档超级详细】
* 很厉害的样子，o(￣▽￣)ｄ  
* 去试玩一下  罒ω罒 
* 问题：不支持openjdk，因为其没有jps,arthas是用jps去找java进程的，需要使用oracle jdk
* 后面的Jstack等参考：https://blog.csdn.net/zbajie001/article/details/80045710

---

### 尝试-快速安装使用

authas是一个jar包，可以直接下载后运行

```shell
wget https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
java -jar arthas-boot.jar -h   # 打印帮助信息
```

就可以启动起来。启动后，authas会自动检测存在的java进程，这时候需要选择你想要诊断的进程，回车即可。

#### 开启使用

```shell
[root@chuangshoubaoceshi01 arthas]# java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.4.4
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 2262 jenkins.war
  [2]: 13819 admin-console-1.1.jar
  [3]: 3103 chuangke-api.jar
2
[INFO] Start download arthas from remote server: https://arthas.aliyun.com/download/3.4.4?mirror=aliyun
[INFO] File size: 11.94 MB, downloaded size: 2.56 MB, downloading ...
[INFO] File size: 11.94 MB, downloaded size: 5.60 MB, downloading ...
[INFO] File size: 11.94 MB, downloaded size: 8.58 MB, downloading ...
[INFO] File size: 11.94 MB, downloaded size: 11.62 MB, downloading ...
[INFO] Download arthas success.
[INFO] arthas home: /root/.arthas/lib/3.4.4/arthas
[INFO] Try to attach process 13819
[INFO] Attach process 13819 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'


wiki      https://arthas.aliyun.com/doc
tutorials https://arthas.aliyun.com/doc/arthas-tutorials.html
version   3.4.4
pid       13819
time      2020-11-24 10:43:32

[arthas@13819]$ dashboard
```

#### dashboard 显示实时数据面板

> | 参数名称 | 参数说明                                |
> | -------- | --------------------------------------- |
> | [i:]     | 刷新实时数据的时间间隔 (ms)，默认5000ms |
> | [n:]     | 刷新实时数据的次数                      |
>
>  **数据说明**
>
> * ID: Java级别的线程ID，注意这个ID不能跟jstack中的nativeID一一对应。
> * NAME: 线程名
> * GROUP: 线程组名
> * PRIORITY: 线程优先级, 1~10之间的数字，越大表示优先级越高
> * STATE: 线程的状态
> * CPU%: 线程的cpu使用率。比如采样间隔1000ms，某个线程的增量cpu时间为100ms，则cpu使用率=100/1000=10%
> * DELTA_TIME: 上次采样之后线程运行增量CPU时间，数据格式为`秒`
> * TIME: 线程运行总CPU时间，数据格式为`分:秒`
> * INTERRUPTED: 线程当前的中断位状态
> * DAEMON: 是否是daemon线程

可以看到，这里会显示出线程(按照cpu占用百分比倒排)、内存(堆空间实时情况)、GC情况等数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201125141126866.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70#pic_center)


#### thread - jvm线程信息

常用情况：

```shell
thread  22   【22为线程id】 将会打印线程运行堆栈信息。
thread --state WAITING  【查看指定状态的线程】
thread -b  【找出当前阻塞其他线程的线程】 
thread -n 3 -i 1000  【列出1000ms内最忙的3个线程栈】
```

#### jvm - 【略】

#### 基础命令

```shell
help——查看命令帮助信息
cat——打印文件内容，和linux里的cat命令类似
pwd——返回当前的工作目录，和linux命令类似
cls——清空当前屏幕区域
session——查看当前会话的信息
reset——重置增强类，将被 Arthas 增强过的类全部还原，Arthas 服务端关闭时会重置所有增强过的类
version——输出当前目标 Java 进程所加载的 Arthas 版本号
history——打印命令历史
quit——退出当前 Arthas 客户端，其他 Arthas 客户端不受影响
shutdown——关闭 Arthas 服务端，所有 Arthas 客户端全部退出
keymap——Arthas快捷键列表及自定义快捷键
```

#### 各种系统信息

```
sysprop - 查看当前JVM的系统属性
sysenv - 查看当前JVM的环境属性
vmoption - 查看，更新VM诊断相关的参数
perfcounter - 查看当前JVM的 Perf Counter信息【性能计数器】
classloader - 查看classloader的继承树，urls，类加载信息
```

#### ognl - 查看静态类等

> 不知道怎么用 ，ε=(´ο｀*)))唉   

#### sc - 用来查询JVM已加载的类信息

```shell
[arthas@13819]$ sc *.*User
com.ibeetl.admin.core.entity.CoreUser
com.ibeetl.admin.core.entity.User
com.ibeetl.app.entity.AppPushUser
com.ibeetl.cms.entity.User
com.ibeetl.cms.web.vo.UserVO
org.apache.xmlbeans.impl.values.JavaIntegerHolder
org.apache.xmlbeans.impl.values.JavaStringHolder
org.apache.xmlbeans.impl.values.TypeStoreUser
org.apache.xmlbeans.impl.values.XmlIntegerImpl
org.apache.xmlbeans.impl.values.XmlObjectBase
org.apache.xmlbeans.impl.values.XmlStringImpl
org.springframework.boot.autoconfigure.security.SecurityProperties$User
Affect(row-cnt:12) cost in 77 ms.
 sc -d  com.ibeetl.admin.core.entity.CoreUser  【查看更具体点的信息】
```

#### sm - 查看已加载类的方法信息

> 查看类的所有方法  -d 可查看参数信息等

```shell
[arthas@13819]$ sm  com.ibeetl.admin.core.entity.CoreUser
com.ibeetl.admin.core.entity.CoreUser <init>()V
com.ibeetl.admin.core.entity.CoreUser setState(Ljava/lang/String;)V
com.ibeetl.admin.core.entity.CoreUser getPassword()Ljava/lang/String;
。。。。。
Affect(row-cnt:25) cost in 14 ms.
```

#### jad - 反编译指定已加载类的源码

```shell
[arthas@13819]$ jad  com.ibeetl.admin.core.entity.CoreUser

ClassLoader:
+-org.springframework.boot.loader.LaunchedURLClassLoader@3f91beef
  +-sun.misc.Launcher$AppClassLoader@5c647e05
    +-sun.misc.Launcher$ExtClassLoader@5a10411

Location:
file:/u01/csb/sys/admin-console-1.1.jar!/BOOT-INF/lib/admin-core-1.1.jar!/

package com.ibeetl.admin.core.entity;

import com.fasterxml.jackson.annotation.JsonIgnore;
import com.ibeetl.admin.core.annotation.Dict;
。。。。。

public class CoreUser
extends BaseEntity {
。。。。。。
```

#### monitor - 方法执行监控

对匹配 `class-pattern`／`method-pattern`的类、方法的调用进行监控。直到用户输入 `Ctrl+C` 为止。

```shell
[arthas@13819]$  monitor -c 5 com.ibeetl.admin.core.entity.CoreUser
The argument 'method-pattern' is required, description: Method of Pattern Matching
[arthas@13819]$  monitor -c 5 com.ibeetl.admin.core.entity.CoreUser
[arthas@13819]$  monitor -c 5 com.ibeetl.admin.core.entity.CoreUser getPassword
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 137 ms, listenerId: 1
 timestamp                   class                                     method                                   total         success       fail          avg-rt(ms)    fail-rate
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 2020-11-24 15:36:57         com.ibeetl.admin.core.entity.CoreUser     getPassword                              1             1             0             0.64          0.00%
```

> 确实可以看到，基本事实的调用情况，这里统计周期为5秒。

#### watch- 方法执行数据观测

查看函数的参数、返回值、异常信息，如果有请求触发，就回打印对应的数据。

>  这个方法参数好多，可以监控方法各个情况的数据。
>
> 如过需要按照耗时进行过滤，需要加入： '#cost>200' 代表耗时超过200ms的才会打印出来。

```shell
[arthas@13819]$ watch  com.ibeetl.admin.core.entity.CoreUser getPassword
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 53 ms, listenerId: 2
ts=2020-11-24 15:45:46; [cost=0.056363ms] result=@ArrayList[
    @Object[][isEmpty=true;size=0],
    @CoreUser[com.ibeetl.admin.core.entity.CoreUser@70f4eb5a],
    @String[123456],
```

#### trace - 方法内部调用路径

方法内部调用路径，并输出方法路径上的每个节点上耗时

`trace` 命令能主动搜索 `class-pattern`／`method-pattern` 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

```shell
[arthas@13819]$ trace  com.ibeetl.admin.core.entity.CoreUser getPassword
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 58 ms, listenerId: 3
`---ts=2020-11-24 15:52:15;thread_name=http-nio-8111-exec-7;id=1c;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@19beff11
    `---[0.05302ms] com.ibeetl.admin.core.entity.CoreUser:getPassword()
# 可以看到，调用这个的方法是最后这个 ↑↑↑↑↑↑↑↑↑
```

> 使用-j (忽略jdk method trace)、'#cost>10'(过滤耗时时间) -n (执行次数)

#### stack - 方法被调用的调用路径

> 很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多，或者你根本就不知道这个方法是从那里被执行了，此时你需要的是 stack 命令。

输出当前方法被调用的调用路径,相当的完整，从线程run开始的，所有调用路径，o(￣▽￣)ｄ 

```shell
[arthas@13819]$ stack  com.ibeetl.admin.core.entity.CoreUser getPassword -n 1
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 51 ms, listenerId: 5
ts=2020-11-24 15:58:19;thread_name=http-nio-8111-exec-9;id=1e;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@19beff11
    @com.ibeetl.admin.core.entity.CoreUser.getPassword()
        at sun.reflect.NativeMethodAccessorImpl.invoke0(NativeMethodAccessorImpl.java:-2)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    。。。。。。。。
        at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
        at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:790)
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1459)
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
# 太多了，中间就略过了 
```

#### tt - 执行方法观测

方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测

> `watch` 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。
>
> 这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。

```shell
[arthas@13819]$ tt  com.ibeetl.admin.core.entity.CoreUser getPassword -t -n 3
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 58 ms, listenerId: 7
 INDEX      TIMESTAMP                   COST(ms)      IS-RET      IS-EXP     OBJECT               CLASS                                     METHOD
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 1000       2020-11-24 17:15:48         0.092734      true        false      0x44e5c9a1           CoreUser                                  getPassword
 1001       2020-11-24 17:15:53         0.02218       true        false      0x322b3578           CoreUser                                  getPassword
 1002       2020-11-24 17:15:56         0.028293      true        false      0x4e46b99c           CoreUser                                  getPassword
Command execution times exceed limit: 3, so command will exit. You can set it with -n option.
```

##### 重做一次调用

> 当你稍稍做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。

> tt -s 'method.name=="primeFactors"'   : 选出 `primeFactors` 方法的调用信息

```shell
# 显示历史 
[arthas@13819]$ tt -l 
 INDEX      TIMESTAMP                   COST(ms)      IS-RET      IS-EXP     OBJECT               CLASS                                     METHOD
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 1000       2020-11-24 17:15:48         0.092734      true        false      0x44e5c9a1           CoreUser                                  getPassword
 1001       2020-11-24 17:15:53         0.02218       true        false      0x322b3578           CoreUser                                  getPassword
 1002       2020-11-24 17:15:56         0.028293      true        false      0x4e46b99c           CoreUser                                  getPassword
Affect(row-cnt:3) cost in 1 ms.
# 查询与重做
[arthas@13819]$ tt -i 1000
 INDEX         1000
 GMT-CREATE    2020-11-24 17:15:48
 COST(ms)      0.092734
 OBJECT        0x44e5c9a1
 CLASS         com.ibeetl.admin.core.entity.CoreUser
 METHOD        getPassword
 IS-RETURN     true
 IS-EXCEPTION  false
 RETURN-OBJ    @String[123456]
Affect(row-cnt:1) cost in 0 ms.
[arthas@13819]$ tt -i 1000 -p
 RE-INDEX      1000
 GMT-REPLAY    2020-11-24 17:23:23
 OBJECT        0x44e5c9a1
 CLASS         com.ibeetl.admin.core.entity.CoreUser
 METHOD        getPassword
 IS-RETURN     true
 IS-EXCEPTION  false
 COST(ms)      0.123085
 RETURN-OBJ    @String[123456]
Time fragment[1000] successfully replayed 1 times.
```

---

### 小结

官方的文档灰常的详细实用，上面的方法，比较的常用，并且只是简单的试了一下。

还有好多的参数玩法是要去看官方文档再去深入玩耍的!!!

而关于这个Arthas不支持open jdk 的问题。好像没法解决啊！ 只能去看看另一个工具：Jstack 的了。

关于Arthas Tunnel 这个的远程管理，就有需要再看了。毕竟分布集群的情况下才会用到。

2020-11 小杭 ヽ(￣ω￣(￣ω￣〃)ゝ 

---

### 关于Jstack - 查看堆栈信息

> jstack能得到运行java程序的java stack和native stack的信息。【堆栈信息】

#### 试试

#### 先找找这东西在哪里

```shell
[root@localhost bin]# find / -name jstack
/usr/local/java/jdk1.8.0_51/bin/jstack

[root@localhost bin]# ./jstack -h
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)
Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
```

#### 看看java项目的进程：

```shell

[root@localhost bin]# top
top - 10:04:56 up 68 days, 28 min,  1 user,  load average: 0.04, 0.12, 0.22
Tasks: 117 total,   1 running, 116 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.2 sy,  0.0 ni, 99.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8010580 total,  1584152 free,  4268572 used,  2157856 buff/cache
KiB Swap:  4063228 total,  4015568 free,    47660 used.  3350348 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
19715 root      20   0 5773368 1.048g  14280 S   1.3 13.7   1:23.82 java
17985 root      20   0 6081164 816336  14492 S   0.7 10.2  14:34.59 java
 2258 root      20   0 5760104 679220   5012 S   0.3  8.5 116:37.37 java
    1 root      20   0  190800   2956   2032 S   0.0  0.0   0:15.68 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:01.04 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:00.97 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      rt   0       0      0      0 S   0.0  0.0   0:00.02 migration/0
# ps -ef | grep java 用这个也是可以找到要的java进程的
```

#### 查看下对应线程的情况

```shell
[root@localhost bin]# top -Hp 17985
top - 10:23:49 up 68 days, 47 min,  1 user,  load average: 0.00, 0.08, 0.18
Threads:  84 total,   0 running,  84 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.2 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8010580 total,  1591992 free,  4267688 used,  2150900 buff/cache
KiB Swap:  4063228 total,  4015568 free,    47660 used.  3351260 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
18076 root      20   0 6081164 817720  14492 S  1.0 10.2   8:23.61 java
18000 root      20   0 6081164 817720  14492 S  0.3 10.2   0:17.44 java
18075 root      20   0 6081164 817720  14492 S  0.3 10.2   0:11.31 java
17985 root      20   0 6081164 817720  14492 S  0.0 10.2   0:00.00 java
17987 root      20   0 6081164 817720  14492 S  0.0 10.2   0:24.73 java
17988 root      20   0 6081164 817720  14492 S  0.0 10.2   0:05.27 java
# 可以看到 cpu消耗最大的线程  
# printf "%x\n" 18076
# 469c  这个是nid，线程编号了 16进制的
```

#### 最后查看堆栈信息

> 因为信息会很多，还是把查到的数据弄到文件吧，方便后面查找

```shell
[root@localhost bin]# ./jstack 17985 > ./test.log
[root@localhost bin]# vim ./test.log
/nid=0x469c   # 用这个vim的查找，就可以查到对应堆栈信息了
# 数据大概这个样子：
"sentinel-time-tick-thread" #24 daemon prio=5 os_prio=0 tid=0x00007fa46c079000 nid=0x469c sleeping[0x00007fa4c13d1000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at com.alibaba.csp.sentinel.util.TimeUtil$1.run(TimeUtil.java:37)
        at java.lang.Thread.run(Thread.java:745)
```

### 关于Jmap-内存分配【略】

得到运行java程序的内存分配的详细情况。这个有需要再说吧，操作太虎咋了╮(╯_╰)╭ 

### 关于Jstat-资源性能监控【实用】

这是一个比较实用的一个命令，可以观察到classloader，compiler，gc相关信息。可以时时监控资源和性能 

官方文档：https://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html

#### 试试

```shell
[root@localhost bin]# ./jstat -gc 17985 100 5
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
512.0  512.0  298.1   0.0   39424.0   6645.5   263168.0   173533.1  126040.0 119185.4 13912.0 12737.1   1786    9.516   4      0.833   10.350
512.0  512.0  298.1   0.0   39424.0   6645.5   263168.0   173533.1  126040.0 119185.4 13912.0 12737.1   1786    9.516   4      0.833   10.350
512.0  512.0  298.1   0.0   39424.0   6645.5   263168.0   173533.1  126040.0 119185.4 13912.0 12737.1   1786    9.516   4      0.833   10.350
512.0  512.0  298.1   0.0   39424.0   6645.5   263168.0   173533.1  126040.0 119185.4 13912.0 12737.1   1786    9.516   4      0.833   10.350
512.0  512.0  298.1   0.0   39424.0   6645.5   263168.0   173533.1  126040.0 119185.4 13912.0 12737.1   1786    9.516   4      0.833   10.350
# 命令解释 jstat [参数] pid [采样间隔默认单位是毫秒] [样本数]
```

具体各个参数的解释请跳转查看官方文档。

给个可以查的东东有哪些：

```shell
-class：统计class loader行为信息 
-compile：统计编译行为信息 
-gc：统计jdk gc时heap信息 
-gccapacity：统计不同的generations（不知道怎么翻译好，包括新生区，老年区，permanent区）相应的heap容量情况 
-gccause：统计gc的情况，（同-gcutil）和引起gc的事件 
-gcnew：统计gc时，新生代的情况 
-gcnewcapacity：统计gc时，新生代heap容量 
-gcold：统计gc时，老年区的情况 
-gcoldcapacity：统计gc时，老年区heap容量 
-gcpermcapacity：统计gc时，permanent区heap容量 
-gcutil：统计gc时，heap情况 
```

---

### 再小结一下

这几个命令是jdk自带的，open jdk 和oracle jdk 都有的，所有还是需要知道一下的。

关于使用的话，可以简单的查看jvm使用情况，以及线程出问题的排查。

用于项目正式环境排查异常还是挺方便的，如果用不了arthas的情况下。

【简单查看jvm信息，这个比arthas还要简单的。装一下还是很方便的。罒ω罒 】

---
