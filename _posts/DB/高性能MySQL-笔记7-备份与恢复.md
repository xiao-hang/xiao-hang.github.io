## 高性能MySQL-笔记7-备份与恢复

> 简单介绍了一下MySQL的一下备份和复制的情况。

---

### 备份

* 物理备份
  * 直接复制MySQL下的表文件【实际复制的话，要备份其相关文件】
* 逻辑备份
  * 使用`mysqldump`进行逻辑备份
  * 其他工具：`mydumper`
  * `select into outfile....` 语句逻辑备份
  * 有一些注意项：需要标准字符集，不要覆盖文件导出
* 文件系统快照？？
  * 使用LVM进行快照的操作。
  * 由于快照是操作时对原始数据块进行复制的，所以是会影响系统读写的。
  * 注意：快照并不是数据的完整备份，只包含快照之后差异数据块的备份。

---

### 恢复

* 物理备份恢复

  * 呀的，就没说怎么具体操作的  (ノ｀Д)ノ 

* 逻辑备份恢复

  * （这个还是有一些的操作提供参考了，不知道是不是真的可以使用  ╮(╯_╰)╭  就先记一下好了 ）

  * ```mysql
    -- 处理SQL导出的本文件的恢复，直接执行就可以了
    -- 暂时停止对bin-log日志的写入 以提高加载数据速度
    -- ===============   操作   =====================
    set SQL_LOG_BIN = 0 ; 
    SOURCE ***_table_backup.sql;
    set SQL_LOG_BIN = 1 ;
    -- ==============================================
    -- 本以为这样就可以挂机等待 sql 文件如期导入了，但是事与愿违，当过一段时间在打开时发现命令行提示链接超时,等待重新链接。
    -- 这时候需要再执行以下 sql：
    -- set global max_allowed_packet=100000000;
    -- set global net_buffer_length=100000;
    -- set global interactive_timeout=28800000;
    -- set global wait_timeout=28800000;
    -- 以上语句的解释：
    -- max_allowed_packet=XXX 客户端/服务器之间通信的缓存区的最大大小
    -- net_buffer_length=XXX TCP/IP 和套接字通信缓冲区大小,创建长度达 net_buffer_length 的行
    -- interactive_timeout 对后续起的交互链接有效时间
    -- wait_timeout 对当前交互链接有效时间。
    ```

  * ```mysql
    
    -- 处理select into outfile 导出的文件
    -- 使用语句 LOAD DATA INFILE 
    -- 对压缩的数据是用linux管道进行操作，据说会很快
    mkfifo /tmp/payment.fifo
    chmod 666 /tmp/payment.fifo
    gunzup -c /tmp/payment.txt.gz > /tmp/payment.fifo
    
    set SQL_LOG_BIN = 0 ; 
    LOAD DATA INFILE '/tmp/payment.fifo' INTO TABLE ***.payment;
    set SQL_LOG_BIN = 1 ;
    
    -- 书上是有这么个操作的，反正我是没用过的 ╮(╯_╰)╭ 
    ```

---

### 一些的工具

* Percona XtraBacku 的备份工具
* mydumper 多线程备份恢复的工具，用于代替mysqldump
* **mysqldump 是最常用通用的工具！！！**【630，有简单栗子】

---

对，就这么点东西，其他的内容，很多都是推荐工具，不建议使用。需要的情况请去搜索最新工具使用。

---











