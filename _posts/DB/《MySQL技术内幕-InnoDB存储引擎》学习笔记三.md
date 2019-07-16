# 《MySQL技术内幕-InnoDB存储引擎》学习笔记三

[TOC]

---

> 2019-07-06   ╮(╯▽╰)╭ 捡起来，继续学习

## 第5章  索引和算法

> InnoDB存储引擎索引的概述：
>
> InnoDB存储引擎，支持的常见索引：B+树索引（常用的），全文索引，哈希索引（无法干预，自动的）。
> 所以最常用的就是B+树索引了，此索引并不能给定一个键值直接找到具体行，而是只能找到行所在的页，然后读取页数据进行查找的。

---

###  索引的数据结构与状态简述

#### 二叉树 B+树 的介绍

**二叉树：**就是两个叉叉的树结构，依据大小分叉。对应的算法是二分查找法。具体是怎么弄的，略过。

**平衡二叉树【AVL树】：**就是上面的这个二叉树做一下平衡而来的。【定义是，所有节点的两个子树的高度差最大为1】平衡二叉树的查找性能比较高，但不一定是最高的，只是维护的成本很高，每次操作都要左右旋来保证平衡。

**B+树：**也是一种的平衡查找树，简单定义是：数据都在同一层叶子节点上（节点间双向连接，数据至少两条），非叶子节点的层记录的是叶子节点的起始值和指针地址，并且都是按顺序存放的。对于具体的定义，还是Google吧。

#### B+索引树

在InnoDB存储引擎中，数据存储就是以B+树的形式存在的，所以数据即索引，默认最大层数为3层。

##### 然而在这里一颗B+树一般能存多少数据呢？

> 这里，InnoDB最小的存储单位为页，**一个页的大小是16K。**（也就是B+树的节点大小了）
>
> 这里我们先假设B+树高为2，即存在一个根节点和若干个叶子节点，那么这棵B+树的存放总记录数为：根节点指针数*单个叶子节点记录行数。
> 所以：**单个叶子节点（页）中的记录数=16K/1K=16。**（这里假设一行记录的数据大小为1k，实际上现在很多互联网业务数据记录大小通常就是1K左右）。
>
> 接下来就是算算非叶子节点能存放多少指针地址了？
> 假设主键ID为bigint类型，长度为8字节，而指针大小在InnoDB源码中设置为6字节，这样一共14字节，我们一个页中能存放多少这样的单元，其实就代表有多少指针，**即16384/14=1170。那么可以算出一棵高度为2的B+树，能存放1170*16=18720条这样的数据记录。**
>
> 根据同样的原理我们可以算出**一个高度为3的B+树可以存放：1170 * 1170 * 16=21902400条这样的记录。**
>
> **所以在InnoDB中B+树高度一般为1-3层，它就能满足千万级的数据存储**。在查找数据时一次页的查找代表一次IO，所以通过主键索引查询通常只需要1-3次IO操作即可查找到数据。

#### 聚集索引和辅助索引

InnoDB中的B+树索引可以分为：聚集索引和辅助索引，他们都是B+树的，不过聚集索引的叶子节点存放的是完整的数据，而辅助索引存放的则是书签（指向主键索引主键的）。

索引作为一个B+树的存储结构，是逻辑上连续，物理存储上不连续的。

**这里说明一下使用辅助索引的过程**：（聚集索引就是直接就找到对应页了）
使用辅助索引查找数据的时候，InnoDB存储引擎会遍历辅助索引并**通过叶级别的指针获取到指向主键索引的主键**，然后再通过主键索引来找到一个完整的行记录。所以对于一个3层聚集索引+3层辅助索引的表，在使用辅助索引查找数据的时候，会进行6次的逻辑IO访问。

#### 简单索引管理语法

直接看代码就好了：

```sql
--第一种：
CREATE <索引名> ON <表名> (<列名> [<长度>] [ ASC | DESC])
--第二种：
ALTER TABLE + 
ADD INDEX [<索引名>] [<索引类型>] (<列名>,…)    -- 表示在修改表的同时为该表添加索引。
ADD PRIMARY KEY [<索引类型>] (<列名>,…)  --表示在修改表的同时为该表添加主键。
ADD UNIQUE [ INDEX | KEY] [<索引名>] [<索引类型>] (<列名>,…)  --表示在修改表的同时为该表添加唯一性索引。
ADD FOREIGN KEY [<索引名>] (<列名>,…)  --表示在修改表的同时为该表添加外键。
-- 其他的各种的参数 数据库默认的就好  不用管的 默认就很好的了
```

查看索引情况：

```sql
SHOW INDEX FROM <表名> [ FROM <数据库名>]
-- 需要注意的值：Cardinality 表示的是索引中的唯一值的数量 【与数据量越接近越好】
```

##### 关于Cardinality 

**这个表示的是索引中的唯一值的数量 ，对于索引来说`Cardinality / count(1) = 1`是最佳的（就是唯一索引的），越接近越好，就是属于高选择性**【一次选择，可以定位到最小的数据量】，如果这个值很小，辣么可以考虑删除这个索引，因为没啥用了。

这个值的计算与更新：

此值是数据库 随机获取8个数据页进行统计的，所以值并不是准确值，【有参数可以配置采集数量，还是不用去调整的好】。
因为这个值并不是实时更新的，但是查询优化的时候会参考这个值进行使用索引，所以，在非高峰期的时候，可以手动的去更新一下此值：`ANALYZE TABLE TABLE_NAME`。

---

### B+树索引的使用

讲了一些类型的索引，但是没有具体太多的操作技巧，都是理论说明。

##### 联合索引

就是使用的索引是多个字段的，如` ALTER TABLE test ADD KEY (P_1,p_2)`，这种情况下，第一层数据结构是对p_1排序的，在相同P_1的情况下，是里面的P_2是排序的。

> 这里的索引也是B+树，存在的key是以（P_1,P_2）这种存在的，大小的比较是先P_1，在P_2的。所以这里，（1,2）>（2，200）的。如果是3+个列的索引也是如此的。

所以，`select * from test where p_1 = 1 order by p_2 desc limit 100` 这种的语句会使用这个联合索引，因为这个索引中已经对p_2排序了的。

二对于3+个列的联合索引，排序都只是一级对一级的 ，例如：

```sql
ALTER TABLE test ADD KEY (P_1,p_2，P_3);
select * from test where p_1 = xxx order by p_2；  			-- 这个是可以直接通过联合索引的
select * from test where p_1=xxx and p_2 = xxx order by p_3   -- 这个是可以直接通过联合索引的
select * from test where p_1=xxx   order by p_3 			-- 这里是不行的，P_1与P_3没有排序关系的
```

##### 覆盖索引

这个东东，就是如果primary key id 的情况下，在添加一个 key id 的普通辅助索引。

因为聚集主键索引，存的是整条数据，而辅助索引存的只是指针，存储大小明显小很多的。

所以在不查询数据的语句如：count(1)这种的情况下，优化器会选择辅助索引来处理的。

##### 其他优化项

**强制使用索引：FORCE INDEX**

```sql
select * from test FORCE INDEX (索引名) where ........
```

**MRR优化和ICP优化：**

```sql
查询方法：
select @@optimizer_switch 
配置参数：
index_condition_pushdown=on,   
mrr=on,mrr_cost_based=on,   -- 第一个：开启，第二个：是否通过cost based 方式选择（自动选择），
--默认就是这样子的 
```

> 然后解释一下这个两个东东：
>
> MMR：Multi-Range Read 优化，目的是为了减少磁盘的随机访问，大概的操作：就是会对查出来的辅助索引中的主键值先排序在缓存中，然后再顺序的访问聚集索引。
>
> ICP：Index Conition Pushdown：对于多条件的查询，会在索引取出数据的同时进行数据筛选过滤。

**哈希表：** 这个就忽略吧，反正是控制不了的东西╮(╯_╰)╭

---

### 全文索引

简单说明介绍：

> 先来说一下全文检索（Full-Text Search）,这个是对信息中的任意内容查找出来。比较像：`select * from blog where content like '%xxx单词/词语%';`这个样子的查询。真正的使用不是这个样子的。
>
> 全文检索通常使用的倒排索引来实现的，也是一个B+树索引，然后存的key是每一个单词/分词，vaule为存在的位置（文档id【inverted file index】，或者文档id+具体位置【full inverted index】）。

#### InnoDB 全文检索

采用的是full invertedj index的方式，**从MySQL 5.7.6开始，`MySQL`提供了一个内置的全文ngram解析器，支持中文，日文和韩文（CJK）** 。官方说明：<https://dev.mysql.com/doc/refman/5.7/en/fulltext-search.html>

在InnoDB中，为提高全文检索并行性能，存在6张的Auxiliary Table(辅助表)，并且还有一个FTS Index Cache的全文检索索引缓存。（FTS Index Cache 默认大小是32M，可以用innodb_ft_cache_size来调整）

弄个简单的代码说明一些东东：

```sql
--创建表：
CREATE TABLE fts_a(
    -- 下面这个字段的名字 和 类型是固定的，如果不手动添加，系统好像是会自己去生成的
	FTS_DOC_ID BIGINT UNSIGNED AUTO_INCREMENT NOT NULL,   
    body TEXT,
    PRIMART KEY(FTS_DOC_ID)
);
--创建索引
CREATE FULLTEXT INDEX idx_fts ON fts_a(body);
--查看分词信息
SET GLOBAL innodb_ft_aux_table = 'test/fts_a';
select * from information_schema.INNODB_FT_INDEX_TABLE;
--手动删除索引 （因为实际DML操作实际并不会删除索引数据，会在deleted表插入记录）
SET GLOBAL innodb_optimize_fulltext_only=1;
OPTIMIZE TABLE fts_a;    --会重组表数据和索引的物理存储
```

#### 具体使用说明

##### 语法：

```sql
MATCH (col1,col2,...) AGAINST (expr [search_modifier])
search_modifier:
  {
       IN NATURAL LANGUAGE MODE    -- 这个是默认值  
     | IN NATURAL LANGUAGE MODE WITH QUERY EXPANSION
     | IN BOOLEAN MODE
     | WITH QUERY EXPANSION
  }
```

##### 自然语言全文搜索：（默认）

```sql
SELECT * FROM fts_a
        WHERE MATCH (body)
        AGAINST ('xxxxxxx' IN NATURAL LANGUAGE MODE);
--因为是默认参数 所以等同于
SELECT * FROM fts_a  WHERE MATCH (body) AGAINST ('xxxxxxx');
-- 统计结果数量
select count(*) FROM fts_a  WHERE MATCH (body) AGAINST ('xxxxxxx');
select count(if(MATCH (body) AGAINST ('xxxxxxx'),1,null)) as count  FROM fts_a ; --这个速度会更快
--查看数据权重
select *，MATCH (body) AGAINST ('xxxxxxx') as Relevance  FROM fts_a ;
```

##### 布尔全文搜索：（IN BOOLEAN MODE）

使用此修饰符，某些字符在搜索字符串中的单词的开头或结尾处具有特殊含义。

来个栗子看看，再做说明：

```sql
SELECT * FROM fts_a WHERE MATCH (body)
        AGAINST ('+Pease -hot' IN BOOLEAN MODE);
-- 表示的意思是必须有‘Pease’,但没有‘hot’的数据
```

> 可以看到，这种的语法有点像正则的说，不过并没有辣么复杂，规则没有辣么多。

支持的运算符（规则）：

> * \+ 表示必须存在
> * \- 表示必须排除
> * （no operator） 默认情况下表示可选，但如果出现，相关性会更高
> * @*distance* 此运算符`InnoDB`仅适用于表。它测试两个或多个单词是否都在相互指定的距离内开始，用单词测量。 例如，在运算符之前的双引号字符串中指定搜索词`@distanceMATCH(col1) AGAINST('"word1 word2 word3" @8' IN BOOLEAN MODE)`
> * \> 表示出现单词时增加相关性
> * \< 表示出现单词时降低相关性
> * ~ 表示允许出现，出现时负相关性
> * \* 表示任意字符 如：`lik*` 表示已`lik`开头的单词
> * " 表示短语？ 例如， `"test phrase"`匹配`"test, phrase"`。两个分词或者关系。

这些规则的使用，就了解一下就可以了，如果需要使用的时候再细看使用。简单的使用的话，应该没有太大的难度，最多就多调调咯╮(╯_╰)╭

##### 扩展全文搜索：（WITH QUERY EXPANSION）

这个东西怎么说明呢，很难描述啊！！

栗子吧，不好说清楚的东东：

```sql
MATCH (body)  AGAINST ('database');
-- ↑ 上面正常全文搜索的话 会出现所有 body 中有 database 的数据行
MATCH (body)  AGAINST ('database' WITH QUERY EXPANSION );
-- ↑ 开启扩展之后  查询到的就不止是 database  了  ，还会包括 Mysql DB2 等等关于数据库的 行
```

> 这里的意思就是除了查询与关键分词一模一样的以外，还需要查询出隐含关联的信息【比如这里的database，数据库关联的信息就是各种数据库：mysql ,DB2 等等的数据库】

但是这里我开始完全不知道这个是怎么扩展的，中文的意思他是怎么关联呢？

并且，这个功能比较好损性能，还是不使用的好 罒ω罒

---

### 小结一下

关于索引，算法方面其实只要了解一下二叉树和B+树是个什么东东，并且主要的查询特性就可以了，其他的交给MySQL本身的去处理就可以了。

而对于索引带来性能优化：

* 在常用的查询位置加一个复合索引
* 对常用的表的ID主键再建立一个索引【方便管理平台之类分页的查询进行count(1) 】
* 而关于强制索引一般是用不上的，除非某一个大表并且有特定大量查询，除此之外的话还是相信MySQL本身的优化器吧，_(:з」∠)_  
* 全文索引，这里基本上只是简单的使用，【这里无法评价，因为有专门的搜索引擎：solr 、ElasticSearch 、Lucene；这些我都只是听过名字，还是技术盲区。】【有对这块了解大神可以简单的介绍评论一下的 Thanks♪(･ω･)ﾉ】

恩，(⊙o⊙)… 就这些了，这一篇的关系Mysql索引的内容就酱紫了。。。。。。。。

---

2019-07-15  小杭  

这本书的内容，接下来的是：锁、事务、备份、优化的内容了。

ヾ(◍°∇°◍)ﾉﾞ 感觉都是用不上的理论知识，让自己看一些数据的时候想的更深一点【虽然没毛线用处】

![img](file:///C:\Users\ADMINI~1\AppData\Local\Temp\SGPicFaceTpBq\37920\0217FA50.gif)



