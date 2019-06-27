## 《MySQL技术内部-InnoDB存储引擎》学习笔记一

[TOC]

---

> 2019-04-24 开始学习 ε=(´ο｀*)))唉

##### 查看MySQL配置文件

`mysql --help | grep my.cof`

> 多个配置，权重按读取到的最后一个配置文件为准

##### 查看数据库存储路径

`SHOW VARIABLE LIKE 'datadir';`

### 第2章 InnoDB 体系架构

#####  查看InnoDB中的线程信息

查看版本：`show variables like 'innodb_version'`
查看线程配置：`show variables like 'innodb_%io_threads'`

> 这里对应了两个相关的配置；innodb_read_io_threads 和 innodb_write_io_threads ，可以再做优化的时候进行

查看状态：`show engine innodb status`
查看Purge Thread :`show variables like 'innodb_purge_threads'`

> 作为事务提交后的回收udo页的线程，参数：innodb_purge_threads 可配置线程 4；

##### 内存相关的东东

查看缓冲池：
`show variables like 'innodb_buffer_pool_size'   --单位B`
`SELECT @@innodb_buffer_pool_size/1024/1024/1024;--单位G  ` 

这个调整的东西比较多：
来源博客：<https://www.cnblogs.com/wanbin/p/9530833.html>
官方文档：<https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool-resize.html>

> **`innodb_buffer_pool_size`默认大小为128M。当缓冲池大小大于1G时，将`innodb_buffer_pool_instances`设置大于1的值可以提高服务器的可扩展性。**
> 大的缓冲池可以减小多次磁盘I/O访问相同的表数据。在专用数据库服务器上，可以将缓冲池大小设置为服务器物理内存的80%。
>
> 在调整缓冲池的时候的规则：增大或减小缓冲池大小时，将以chunk的形式执行操作。chunk大小由`innodb_buffer_pool_chunk_size`配置选项定义，默认值为128 MB。
> **配置大小必须是`innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances`的倍数。**（如果不是，缓存池大小会被系统默认重置）
>
> PS：
> innodb_buffer_pool_instances：缓冲池实例个数 
> innodb_buffer_pool_chunk_size ：缓冲池实例最小块内存单位，默认值为128 MB。
> innodb_buffer_pool_read_requests :它表示从内存中逻辑读取的请求数。
> innodb_buffer_pool_reads :表示InnoDB缓冲池无法满足的请求数。需要从磁盘中读取。
>
> 配置依据：
> **InnoDB buffer pool 命中率 = innodb_buffer_pool_read_requests / (innodb_buffer_pool_read_requests + innodb_buffer_pool_reads ) * 100** 
> **数据获取：使用`show status like 'innodb_buffer_pool_read%';`**
> 此值低于99%，则可以考虑增加innodb_buffer_pool_size。
>
> 查看数据页（data_page）使用信息：`show global status like '%innodb_buffer_pool_pages%';`
>
> 最后，备注一下如果加大了很多InnoDB缓冲池的情况下，也要考虑加大一下额外内存池。

查看脏页刷新值：
`show variables like 'innodb_io_capacity'    --200 `   ：当IO性能很好的时候可以增加

查看脏页刷新阀值：
`show variables like 'innodb_max_dirty_pages_pct'    --75%脏页时刷新`
自适应刷新配置相关：
`show variables like 'innodb_adaptive%' `

##### 关键特性

* 插入缓存（Insert Buffer）
* 两次写(Doube Weite)
* 自适应哈希索引
* 异步IO
* 刷新邻接页

### 第3章 文件

##### 慢查询日志

查看慢查询是否开启等：`show variables like 'slow_query_log%'` [默认关闭]
查看慢查询时间设置：`show variables like 'long_query_time' 单位秒`

> 其他的一些配置说明：
> slow_query_log=off|on     --是否开启慢查询日志
> slow_query_log_file=filename --指定保存路径及文件名，默认为数据文件目录，hostname-slow.log
> long_query_time=2    --指定多少秒返回查询的结果为慢查询
> log_throttle_queries_not_using_indexes   -- 每分钟允许记录到日志且没有使用索引的sql语句数量

分析日志：
mysqlsla： 书中用的是这个，来分析的
percona-toolkit中的pt-query-digest ： 不知道是啥

设置日志输出格式：
默认是FILE，文件；
修改log_output = 'TABLE' ,则可以记录日志到表 mysql.slow_log 中；【动态且全局的，可以随时配置】

##### 二进制日志 - binary log

默认也是没有开启的 ╮(╯_╰)╭
查看开启状态：`show variables like 'log_bin%'`
开启【参数为动态的】：`SET sql_log_bin=ON/OFF`    
使用mysqlbinlog 工具来查看；

>  开启binary-log 功能配置：
> log-bin=mysql-bin 
> binlog-format=ROW 
> server-id = 1 
> expire-logs-days = 14
> max-binlog-size = 500M
>
> 官方参考参数地址：<https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#replication-optvars-binlog>

binlog_formar参数的东西

> 在使用binlog进行主从同步的时候，如果运行的语句是有**rand、uuid**等随机函数的情况下，会出现数据不一致的情况。【下面ROW和MIXED参数可以解决这个问题的，推荐MIXED-会在这种问题的时候自动变为ROW】
>
> 所以这里有三个参数：
> STATEMENT：--记录的是逻辑SQL，会存在上面这种随机函数的问题。
> ROW：--记录的是表的行更改情况。Innodb事务隔离设置为COMMITTED会有更好的并发性。
> MIXED：--这个模式默认是STATEMENT，在一些特定情况下会自动变成ROW ╮(╯_╰)╭ 建议使用这个

##### InnoDB的表空间文件

InnoDB默认有一个10MB的ibdata1文件【.ibd-这个是默认表空间】,这里有两个参数对它进行配置：

> show variables like 'innodb_data_file_path'; 
> innodb_data_file_path=ibdata1:200M:autoextend    --这里可以配置多个目录以及文件大小和自增
> e.g.：innodb_data_file_path=/db/ibdata1:200M;/db/ibdata2:200M:autoextend
>
> show variables like 'innodb_file_per_table';   --默认启动  每表独立表空间文件

##### InnoDB的重做日志文件

说的很重要的样子，在data目录下面默认的两个文件：ib_logfile0,ib_logfile1；
但是通常只要用默认的配置就可以了，是为了引擎提供可靠服务存在的。

---

到这里为止，都是MYSQL本身的架构与配置，都是系统级别的优化。看看就好 ┐(´∇｀)┌

这个笔记先到这里：2019-06-25 

ε=(´ο｀*)))唉 快学不动了。。。。

（接下来还是要继续更新的 (ง •_•)ง）





