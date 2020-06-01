## 高性能MySQL-笔记1-基础数据获取分析

[TOC]

---

### 第三方存储引擎

* **Infobright：数据仓库**
  * 超高的查询统计优化和压缩比例（10:1~40:1），
  * 引入了列存储方案,
  * 不需要建索引,使用内部知识网格节点。
  * 查询性能高：百万、千万、亿级记录数条件下，同等的SELECT查询语句，速度比MyISAM、InnoDB等普通的MySQL存储引擎快5～60倍。（TB级数据）
  * 不支持数据更新：社区版Infobright只能使用“LOAD DATA INFILE”的方式导入数据，不支持INSERT、UPDATE、DELETE。
  * 不支持高并发：只能支持10多个并发查询。
* **TokuDB** 
  * 专为在写入密集型工作负载上实现高性能而设计，可通过分形树索引实现。
  * 适合像 Zabbix 这种高 INSERT，少 UPDATE 的应用场景

---

### 基准测试

工具：**sysbench**    (工具还有很多的，例如:TPCC-MYSQL )

作用：测试系统的硬件性能，也可以用来对数据库进行基准测试;【P58】

> 感觉上没什么用，通常的性能瓶颈都是在特定表的查询上。

在调整MySQL系统配置【内存池大小等】的时候，可以使用基准测试验证性能提升比例。

---

### 性能剖析

#### **performance_schema**

提供了一种在数据库运行时实时检查server的内部执行情况的方法

> performance_schema通过监视server的事件来实现监视server内部运行情况， “事件”就是server内部活动中所做的任何事情以及对应的时间消耗，利用这些信息来判断server中的相关资源消耗在了哪里?一般来说，事件可以是函数调用、操作系统的等待、SQL语句执行的阶段(如sql语句执行过程中的parsing 或 sorting阶段)或者整个SQL语句与SQL语句集合。事件的采集可以方便的提供server中的相关存储引擎对磁盘文件、表I/O、表锁等资源的同步调用信息。

信息表：【有很多】

> events_statements_current当前语句事件表
> events_statements_history历史语句事件表
> events_statements_history_long长语句历史事件表

记录一些简单使用【未测试】：

> 来源：<http://blog.itpub.net/24930246/viewspace-2644172/>
>
> 打开等待事件的采集器配置项开关，需要修改setup_instruments 配置表中对应的采集器配置项。
>
> ```mysql
> update setup_instruments set enabled = 'YES', timed = 'YES'where name like 'wait%';
> ```
>
> 打开等待事件的保存表配置开关，修改修改setup_consumers 配置表中对应的配置项。
>
> ```mysql
> update setup_consumers set enabled = 'YES' where name like '%wait%';
> ```
>
> 配置好之后，我们就可以查看server当前正在做什么，可以通过查询events_waits_current表来得知，该表中每个线程只包含一行数据，用于显示每个线程的最新监视事件(正在做的事情);

我们大多数时候并不会直接使用performance_schema来查询性能数据，而是使用**sys schema**下的视图代替;

#### MySQL percona-toolkit [最常用工具]

> 官网：<https://www.percona.com/doc/percona-toolkit/2.2/index.html>
> 博客参考：
> <https://blog.csdn.net/wanbin6470398/article/details/83178755>
> <https://blog.csdn.net/demonson/article/details/87270061>
>
> 超级好像的工具的感觉，看了下提供的一些命令，相当的丰富。
>
> **这个工具，估计真的需要好好研究使用一下，很装逼的样子(▼へ▼メ)**

作用：Percona Toolkit是Percona（<http://www.percona.com/>）的支持人员使用的高级命令行工具的集合，以执行各种MySQL和系统任务，这些任务很难或太复杂而无法手动执行。

> 常用的功能介绍【部分】：【https://blog.csdn.net/demonson/article/details/87270061】
>
> * **pt-archive**：数据库归档
> * **pt-duplicate-key-checker**：用来检查mysql表是否有重复的索引和外键。
> * **pt-mysql-summary**：MySQL的状态和配置统计信息。
> * **pt-query-digest**：分析mysql慢查询的一个工具。※
> * pt-stalk : 监控系统信息，可配置各种阀值
> * 主从数据库相关
> * 权限，磁盘IO信息等等

#### Oprofile

是linux上的性能监测工具，通过CPU硬件提供的性能计数器对事件进行采样，从代码层面分析程序的性能消耗情况，找出程序性能的问题点。

> 用于，数据库性系统性能使用占比。来分析，性能问题的瓶颈。

#### iostat

不知道是啥，书里是用来看io流情况的。╮(╯_╰)╭ [P100] 

---

### Mysql查询分析

工具：慢查询日志+MySQL percona-toolkit - pt-query-digest

作用：查询整个MySQL的查询性能，并定位需要优化的查询。

对于单条SQL的分析，使用Nacicat Premium 工具里面的分析就很全面了。

> 查看慢日志统计：
>
> ```shell
> 按秒：
> awk '/^# Time:/{print $3,$4,c;c=0}/^# User/{c++}' mysql-slow-3306.log
> 
> 按天：
> grep  Time: mysql-slow-3306.log | awk '{print $3}'|awk '{a[$1]++}END{for (j in a) print j,": " a[j]}' | sort -nrk2 -t: 
> 
> 按分钟：
> grep  Time: mysql-slow-3306.log | awk '{print $3,$4}'|awk -F : '{print $1,$2}' | awk -F : '{a[$1]++}END{for (j in a) print j,": " a[j]}' | sort -nrk2 -t:
> 
> 按小时：
> grep  Time: mysql-slow-3306.log | awk '{print $3,$4}'|awk -F : '{print $1}' | awk -F : '{a[$1]++}END{for (j in a) print j,": " a[j]}' | sort -nrk2 -t:
> ```
>
> 查看消耗的内存(需要分配的内存)：
>
> ```mysql
> select event_name,SUM_NUMBER_OF_BYTES_ALLOC/1024/1024 from memory_summary_global_by_event_name order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 10;
> ```
>
> 

---

### 参考

* MySQL各种Tips【各种常见问题】： <https://www.cnblogs.com/zhoujinyi/archive/2013/01/12/2857408.html>

---



