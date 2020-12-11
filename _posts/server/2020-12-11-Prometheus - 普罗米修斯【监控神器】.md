---
layout: post
#标题配置
title:   Prometheus - 普罗米修斯【监控神器】
#时间配置
date:   2020-12-11 12:00:00 +0800
#大类配置
categories: 服务组件
#小类配置
tag: Prometheus
---

* content
{:toc}




### 计划  及  参考文章

* Prometheus  【监控神器普罗米修斯】   

  * Prometheus监控报警系统：https://www.cnblogs.com/chenqionghe/p/10494868.html

  * 官方文档：https://prometheus.io/docs/introduction/overview/

  * 简书：https://www.jianshu.com/p/93c840025f01

  * Prometheus+ Grafana ：https://www.cnblogs.com/sunyllove/p/11212835.html 

  * [SpringBoot 1.4.x 1.x + Prometheus + Granfan 监控体系搭建](https://www.cnblogs.com/yidiandhappy/p/13957200.html)

  * 入门教程：https://www.jianshu.com/p/28f71ea99e3a

  * 报警插件：[Alertmanager](https://github.com/prometheus/alertmanager)

  * > 如何将日志输入Prometheus？【官方说明】
    >
    > 简短的回答：不要！请改用[ELK堆栈之](https://www.elastic.co/products)类的东西。
    >
    > 更长的答案：Prometheus是一个收集和处理指标的系统，而不是事件记录系统。Raintank博客文章 [Logs and Metrics and Graphs，Oh My！](https://blog.raintank.io/logs-and-metrics-and-graphs-oh-my/) 提供有关日志和指标之间差异的更多详细信息。

  * 建议您使用[Grafana](https://prometheus.io/docs/visualization/grafana/)进行生产。也有[控制台模板](https://prometheus.io/docs/visualization/consoles/)。

  * Pushgateway ： https://www.cnblogs.com/shhnwangjian/p/10706660.html

  * mtail ： 可以从应用程序日志中提取指标，并将其导出到时间序列数据库或时间序列计算器中，以便配置警报和仪表盘的工具。

  * 操作指南：https://www.bookstack.cn/read/prometheus-book/README.md

  * 《Prometheus监控实战》第9章 日志监控: https://cloud.tencent.com/developer/article/1556769

  * 容器领域的十大监控系统对比： https://zhuanlan.zhihu.com/p/57740192

---

### 介绍

*Prometheus* 是一个开源的系统监控和警报工具包。非常适合记录时间序列数据，比如可以记录机器CPU、Memory的使用情况；也可以在微服务中收集各个维度的信息。

> 而，商圈项目那边，用的确实用这个来收集日志数据，进行处理的。
> 感觉与官方的推荐相悖啊
> ε=(´ο｀*)))唉

关于它的特点就不介绍了，官方有各种好听的说法，我觉得重点还是在与他是个生态系统。

其本身，就是一个存时间序列的数据集【数据库】，支持拉取数据，以及拥有外围组件生态的东东。

> 简单说明下特点：
> 多种图表和仪表盘，灵活的查询语言(PromQL)，支持push数据到中间件(pushgateway)。

**Prometheus生态系统由多个组件构成**，其中多是可选的，根据具体情况选择

* **Prometheus server** - 收集和存储时间序列数据
* **client library** - 用于client访问server/pushgateway
* **pushgateway** - 对于短暂运行的任务，负责接收和缓存时间序列数据，同时也是一个数据源
* exporter - 各种专用exporter，面向硬件、存储、数据库、HTTP服务等
* **alertmanager - 处理报警**
* **Grafana - 度量分析和可视化工具**
* 其他各种支持的工具

官方给的架构：

![img](https://img-blog.csdnimg.cn/img_convert/31ddc9bc87ba697e4bcc13862591521f.png)

差不多就是这个样子了。

说法上，用来监控各个项目服务情况，JVM的使用情况等的性能数据，很是合适的。

---

### 安装Promotheus

为了方便，当然是在docker上搞咯；

先准备个配置：

```shell
vim /u01/promotheus/promotheus.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']
```

然后创建开启容器：

```shell
docker run \
    -p 9090:9090 \
    -v /u01/promotheus/promotheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

对，这个样子就可以了。

然后访问：[http://192.168.1.180:9090](http://192.168.1.180:9090/) ，就可以看到东东了。【当然要翻译了，英文看不懂 ╮(╯_╰)╭】

![](https://img-blog.csdnimg.cn/20201202144145985.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


### exporter 各种导出器 -> Promotheus

> 先搞点数据进去看看，会是什么样子的。
>
> prometheus官方下载地址：https://prometheus.io/download/  
> go官方下载地址：https://golang.org/dl/
> 64位下载包地址（此地址下载比较快）：https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz 

有好多的导出器，先搞个机器本身数据的试试：node_exporter - 机器指标导出器；

>  node_exporter配置参考：
>
>  安装配置：https://www.jianshu.com/p/7bec152d1a1f
>  数据监控配置：https://www.cnblogs.com/minseo/p/13403478.html  【很实用】
>  exporter种类参考：https://www.bookstack.cn/read/prometheus-book/exporter-what-is-prometheus-exporter.md 
>
>  完成后，请使用上面这个数据监控配置，查询数据。语法我还不会，先copy用着吧。

#### 安装配置node_exporter

```shell
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xvf node_exporter-1.0.1.linux-amd64.tar.gz
[root@localhost node_exporter-1.0.1.linux-amd64]# nohup ./node_exporter
nohup: 忽略输入并把输出追加到"nohup.out"
[root@localhost node_exporter-1.0.1.linux-amd64]# ps -ef |grep node
root     15542 15184  0 14:11 pts/0    00:00:00 ./node_exporter
root     15554 15184  0 14:11 pts/0    00:00:00 grep --color=auto node
curl http://192.168.1.180:9100/metrics 
...... # 这里就可以看到一堆的信息了
```

#### 配置Promotheus 拉取数据

```shell
vim /u01/promotheus/promotheus.yml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'linux-180'
    static_configs:
      - targets: ['192.168.1.180:9100']
        labels:
          instance: node1
:qw
# 然后去重启Promotheus ，我就直接docker重启了 
```

然后再访问就有数据啦！！！！

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144245612.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144256606.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


> Promotheus 服务的东东，先就这样吧。可以各种操作统计数据，然后懒得试了，以后再研究了。
>
> 这里就玩个表面的东东了先   ┐(ﾟ～ﾟ)┌  

### 安装Grafana

> Grafana官网（https://grafana.com/）  
>
> 安装什么的，查看官方的就行了，无论在什么平台的好像都有详细的文档的。o(￣▽￣)ｄ

```shell
docker run  -d --name grafana -p 3000:3000  grafana/grafana
# http://192.168.1.180:3000/    默认 admin：admin
```

然后参考文章：https://www.cnblogs.com/imyalost/p/9873641.html  进行玩耍。

或者，像我一样随便玩 (～￣▽￣)～  

创建好数据源，创建仪表盘，然后把在上面用的查询语句，copy进去就可以了：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120214431594.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


**Grafana-的好用处就是，花里胡哨。可以弄出来灰常好看很有噱头的仪表盘主页 罒ω罒** 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144322754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


#### 花里胡哨的仪表盘 罒ω罒

> 仪表盘导入教程参考：https://www.cnblogs.com/imyalost/p/9873641.html

* 数据库：https://github.com/percona/grafana-dashboards
* 官方就有很多的仪表盘：BashBoard地址：[BashBoard](https://grafana.com/dashboards?dataSource=influxdb) 
  * 只要一个id就可以完成仪表盘导入了 罒ω罒 
  * 而且足够的丰富，足够的花里胡哨。(～￣▽￣)～ 
* Node Exporter Full：https://github.com/rfrail3/grafana-dashboards/blob/master/prometheus/node-exporter-full.json

这才是花里胡哨：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144336620.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


---

---

### 日志监控 -> Promotheus

参考文章：https://cloud.tencent.com/developer/article/1556769

可选方案：

* grok_exporter（https://github.com/fstab/grok_exporter）
* mtail的Google实用程序（https://github.com/google/mtail）
  * 配置：https://blog.csdn.net/bluuusea/article/details/105508897

**选用mtail**，官方说明就是：从应用程序日志中提取白盒监视数据以收集到时间序列数据库中；【完全符合的】

#### mtail -测试

##### 安装配置mtail

```shell
wget https://github.com/google/mtail/releases/download/v3.0.0-rc38/mtail_v3.0.0-rc38_linux_amd64
chmod 777 mtail_v3.0.0-rc38_linux_amd64
mv mtail_v3.0.0-rc38_linux_amd64 mtail
# sudo cp mtail /usr/local/bin 这个我就不操作到系统用户下了
 ./mtail --version
mtail version v3.0.0-rc38 git revision 0601c73197c5e314dcf3e27218de8f8ad5a83691 go version go1.15.2 go arch amd64 go os linux

# 测试  
 touch line_count.mtail
 vim line_count.mtail
# 去这里找一个合适的配置     https://github.com/google/mtail/tree/master/examples

./mtail -logtostderr --progs /u01/mtail/line_count.mtail  --logs '/u01/happyPay/admin/logs/admin-log/*.log'
# nohup ./mtail -port 3903 -logtostderr -progs /u01/mtail/line_count.mtail -logs '/u01/happyPay/admin/logs/admin-log/*.log' >/dev/null 2>&1 &

# 访问http://192.168.1.194:3903/ 就可以看到信息了
```

虽然看不懂，但是感觉有点内容了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144424479.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


##### 配置普罗米修斯

```shell
  - job_name: 'mtail-194'
    static_configs:
      - targets: ['192.168.1.194:3903'] 
        labels:
          instance: log-dev
```

重启，加载配置之后，可以看到抓取点已经配置进去了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202144356862.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RpYW5YdWVXdQ==,size_16,color_FFFFFF,t_70)


> **虽然这里已经有数据了，但是这些数据不能用啊，都是些没什么用的量，比如日志文件数量啥的。**
>
> **我么真正想要的是：分析日志的数据，各个接口等的东西。**
>
> **这结果算是成功，后续还要自己写分析日志的配置。**

#### 调整mtail配置-日志分析

由于各个项目的日志格式不同，这里如有想做日志分析，辣么在mtial这里就需要用正则表达式进行创建对应的指标了。

可以分组的，依据各个内容参数。【如果日志格式确定的话，多项目也可同时使用】

##### 栗子-统计报错原因

mtail配置文件：

```shell
counter error_type by type

/^.+Exception:(?P<type>.*)/ {
  error_type[$type]++ 
}
```

结果数据：

```shell
error_type{prog="line_count.mtail",type=" 用户名或密码错误"} 2
error_type{prog="line_count.mtail",type=" 用户名或密码错误] with root cause"} 1
```

**恩，差不多就这样了，学不动了 ε=(´ο｀*)))唉**

#### 日志监控小结

> 这个日志监控，如果是项目日志还是用ELK的好啊！
> 中间件的日志：比如nginx，是可以使用这个的，毕竟有装本的exporter.【日志格式一般是标准的】
>
> 一定要做的话，需要在mtail收集的地方，就要配置各种的指标了，对运维还是有点要求和工作量的。
>
> 对比于ELK的全量文件导入，进行全文检索，进行后续分析。这个的动态可配置性就差了很多了。

---

### 小结

**Prometheus**这一套监控系统，**在对硬件以及中间件的监控上灰常的强大，且配置使用成本低。**
【硬件：本身linux机子；中间件：数据库，nginx等】

**需要使用各种官方提供的导出器【exporter】+ Grafana的官方仪表盘，这种组合很炫，还很轻松。**

如果是项目日志监控，使用起来需要在收集器进行指标分析等的了。【项目日志标准话是在哪都推荐的】
这样，在Grafana才可以进行再次分析展示出来。

**所以日志监控，还是建议使用ELK啊。**。。ε=(´ο｀*)))唉  虽然学习成本比较高，但是运维轻松啊！【使用交给项目人员自己查询即可】

#### 其他

至于，Prometheus还有其他的组件，比如重要的报警插件：[Alertmanager](https://github.com/prometheus/alertmanager)  等，已经学不动了。

以后有机会再研究了。。。。。。。。

---

小杭 2020-12  (((┏(;￣▽￣)┛装完逼就跑 

---
