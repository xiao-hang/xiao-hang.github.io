## MySQL查询系统信息

### 查询系统状态信息

使用：mysqladmin-ext的命令

#### 作用

作为MySQL使用SHOW GLOBAL STATUS 查看数据库使用状态的脚本。

便于分析数据库使用情况：查询，并发，连接数等等。

这里作为脚本进行格式化数据格式进行查询。【方便查看，原始数据太多了】

#### 脚本1

> MySQL系统运行状态实时监控，已测试 可用,数据比简陋

```shell
#!/bin/bash

mysqladmin -uroot -p{password} ext -i1 | awk 'BEGIN{lswitch=0;
print "|QPS        |Commit     |Rollback   |TPS        |Threads_con  |Threads_run |";
print "------------------------------------------------------------------------------";}
$2 ~ /Queries$/ {q=$4-lq; lq=$4;}
$2 ~ /Com_commit$/ {c=$4-lc; lc=$4;}
$2 ~ /Com_rollback$/ {r=$4-lr; lr=$4;}
$2 ~ /Threads_connected$/ {tc=$4;}
$2 ~ /Threads_running$/ {tr=$4;
if (lswitch==0)
{
  lswitch=1;
  count=0;
} else {
  if (count>10) {
    count=0;
    print "------------------------------------------------------------------------------";
    print "|QPS        |Commit     |Rollback   |TPS        |Threads_con  |Threads_run |";
    print "------------------------------------------------------------------------------";
  } else {
    count+=1;
    printf ("|%-10d |%-10d |%-10d |%-10d |%-12d |%-12d| \n", q,c,r,c+r,tc,tr);
  }
}
}'
```

---

#### 脚本2

> 扩展更多数据的情况，已测试 可用，数据详细完善

```shell
mysqladmin -P3306 -uroot -p -h127.0.0.1 -r -i 1 ext |\
awk -F"|" \
"BEGIN{ count=0; }"\
'{ if($2 ~ /Variable_name/ && ((++count)%20 == 1)){\
    print "----------|---------|--- MySQL Command Status --|----- Innodb row operation ----|-- Buffer Pool Read --";\
    print "---Time---|---QPS---|select insert update delete|  read inserted updated deleted|   logical    physical";\
}\
else if ($2 ~ /Queries/){queries=$3;}\
else if ($2 ~ /Com_select /){com_select=$3;}\
else if ($2 ~ /Com_insert /){com_insert=$3;}\
else if ($2 ~ /Com_update /){com_update=$3;}\
else if ($2 ~ /Com_delete /){com_delete=$3;}\
else if ($2 ~ /Innodb_rows_read/){innodb_rows_read=$3;}\
else if ($2 ~ /Innodb_rows_deleted/){innodb_rows_deleted=$3;}\
else if ($2 ~ /Innodb_rows_inserted/){innodb_rows_inserted=$3;}\
else if ($2 ~ /Innodb_rows_updated/){innodb_rows_updated=$3;}\
else if ($2 ~ /Innodb_buffer_pool_read_requests/){innodb_lor=$3;}\
else if ($2 ~ /Innodb_buffer_pool_reads/){innodb_phr=$3;}\
else if ($2 ~ /Uptime / && count >= 2){\
  printf(" %s |%9d",strftime("%H:%M:%S"),queries);\
  printf("|%6d %6d %6d %6d",com_select,com_insert,com_update,com_delete);\
  printf("|%6d %8d %7d %7d",innodb_rows_read,innodb_rows_inserted,innodb_rows_updated,innodb_rows_deleted);\
  printf("|%10d %11d\n",innodb_lor,innodb_phr);\
}}'
```

---

### 查询各个状态的线程数量

 使用： SHOW PROCESSLIST 命令 [P91]

#### 作用

查询数据库各个状态下的线程数，用以判断是否存在线程堆积等的情况。

#### 脚本

```shell
mysql -u root -p{password} -e 'SHOW PROCESSLIST\G' | grep State: | sort | uniq -c | sort -rn 
```

---

### 其他

查看my.cnf文件地址：mysql --help | grep 'Default options' -A 1

---

