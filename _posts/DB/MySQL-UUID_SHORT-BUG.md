---
layout: post
#标题配置
title:  MySQL-使用UUID_SHORT( ) 的问题
#时间配置
date:   2019-04-04 19:00:00 +0800
#大类配置
categories: BUG
#小类配置
tag: MySQL
---

* content
{:toc}



### 问题说明

表`app_msg`的主键id 设置的类型为：**bigint 20** 

使用插入语句：` INSERT INTO `app_msg`(`id`,...) VALUES  (UUID_SHORT(),...)`

然而系统报错：`[Err] 1264 - Out of range value for column 'id' at row 1`

> 在测试环境使用时没有问题，但是在准生产数据库使用时报错了

### 简单分析解决

#### 分析

在两个数据库查看UUID_SHORT()生成的情况：

```mysql
select UUID_SHORT()   
--结果1：26047177025388691      这里生成了17位的UUID_SHORT
--结果2：18040425909390934036   这里生成了20位的UUID_SHORT
```

> 结果1为17位没问题，结果2位20位导致报错的时候超过最大值

#### 解决

这里可以知道原因是在字段类型上面，**bigint 20** 对应的类型是 **long long** 类型 【长度为：（-2^63 ~ 2^63-1） 10^18  19位数字】；

而UUID_SHORT() 返回的是 **unsigned long long** 类型【长度为：（0 ~ 2^64-1） 10^19  20位数字】

所以，原因是在MySQL设置的时候没有**勾选无符号****这个选项导致的，勾选上就解决了。╮(╯▽╰)╭ 就是这么的简单的地方。

> 临时解决方式，使用时间错手动生成了一个19位以内的随机id：
>
> ```mysql
> FLOOR(REPLACE(unix_timestamp(current_timestamp(3)),'.',''))*10000+FLOOR(RAND()*10000)
> ```

### 认真查了一些详细的资料

#### 官方资料

> 版本：MySQL 5.7 

**UUID_SHORT()**

将“ 短 ”通用标识符作为**64位无符号整数**返回。返回的值 UUID_SHORT()与UUID()函数返回的字符串格式128位标识符 不同，并且具有不同的唯一性属性。UUID_SHORT()如果满足以下条件，则保证值 是唯一的：

* 在server_id当前服务器的值介于0和255之间，您的设置主从服务器中是唯一的
* 您不会在mysqld restarts 之间设置服务器主机的系统时间
* UUID_SHORT()在mysqld重启 之间， 你平均每秒调用的次数少于1600万次

**该UUID_SHORT()返回值的构造是这样的：**

```mysql
  (server_id & 255) << 56
+ (server_startup_time_in_seconds << 24)
+ incremented_variable++;
```

```mysql
mysql> SELECT UUID_SHORT();
        -> 92395783831158784
```

> UUID_SHORT() 不适用于基于语句的复制。

按照以上官方的说法，UUID_SHORT返回的值为64位无符号整数，也就是unsigned long long类型。而且，由于使用到了时间搓，这个的初始值会很大（通常都会到17位数字）。

但是，这并不保证其不会返回18，19，20位的数据，只能保证在2^64-1以内（最大达到20位数字）；

> 目前没有查到说可以调整或初始设置UUID_SHORT() 的配置



---

2019-04-04 小杭  ミ(・・)ミ





