---
layout: post
#标题配置
title:   MySQL随机函数RAND() 概率不稳定
#时间配置
date:   2019-03-13 19:00:00 +0800
#大类配置
categories: BUG
#小类配置
tag: MySQL
---

* content
{:toc}




### 说明一下问题

在一下查询的时候，虽然都是使用RAND()<(20/100)进行随机筛选数据，但是前者获取的比例并不在20%左右，而是在5%左右。

唯一不同的地方在与子查询里面的条件`a.pos_sn = b.pos_sn `和 `a.merchant_code = b.merchant_code`,并且其他条件都一样。

【这两个条件不影响子查询的结果，子查询的结果是一模一样的】

### 代码 一

```sql
  explain 
select  p_i.pay_product as pay_product,
    				p_i.merchant_code as merchant_code ,
    				p_i.merchant_mobile as mobile,
    				CONCAT(FLOOR(200000 + (RAND() * 700000)),"******",FLOOR(1000 + (RAND() * 9000)) ) as trade_card_no,
--     				FLOOR(#directAmount# + (FLOOR(RAND() * #indirectAmount#)*100)) as trade_amount,
    				DATE_FORMAT(CONCAT(DATE_SUB(curdate(),INTERVAL 1 DAY),' ',LPAD(FLOOR(0 + (RAND() * 23)),2,0),':',LPAD(FLOOR(0 + (RAND() * 59)),2,0),':',LPAD(FLOOR(0 + (RAND() * 59)),2,0)),'%Y-%m-%d %T') as trade_time,
    				REPLACE(UUID(),"-","")  as trade_no,
    				1 as trade_rate,
    				3 as brokerage_amount,
    				3 as source_type
    from order_info o_i
    left join (
    			select a.order_id as order_id,IFNULL(sum(b.trade_amount),0) as sumamount
    			from pos_info a
    			left join pos_trade_deal b on a.merchant_code = b.merchant_code  --出问题的条件
        					and b.trade_time > ADDDATE(CURRENT_DATE(),-1) 
        					and b.trade_time < CURRENT_DATE()
    			where a.status = 2 and a.order_id != 0 
    			GROUP BY a.order_id
    ) t_a on t_a.order_id = o_i.id
    left join pos_info p_i on p_i.order_id = o_i.id 
    where o_i.del = 0 and o_i.project_status = 3 
    	and t_a.sumamount = 0   and RAND()<(20/100)
    	and p_i.id is not null
```

 数据量为450   概率结果为 10-50之间跳动

分析结果：

```
id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
1	PRIMARY	<derived2>	ALL	2	50	Using where
1	PRIMARY	o_i	eq_ref	PRIMARY,idx_pou	PRIMARY	8	t_a.order_id	1	5	Using where
1	PRIMARY	p_i	ref	PRIMARY,idx_pos_opu_id	idx_pos_opu_id	8	t_a.order_id	222042	90	Using index condition
2	DERIVED	a	range	idx_pos_opu_id	idx_pos_opu_id	8	2	10	Using index condition; Using where; Using temporary; Using filesort
2	DERIVED	b	range	pos_sn,idx_trade_time	idx_trade_time	6	1	100	Using where; Using join buffer (Block Nested Loop)
```

**1	PRIMARY	<derived2>	ALL	2	50	Using where**

### 代码二

>  这个代码是没问题的，可以使用

```sql
  explain 
select  p_i.pay_product as pay_product,
    				p_i.merchant_code as merchant_code ,
    				p_i.merchant_mobile as mobile,
    				CONCAT(FLOOR(200000 + (RAND() * 700000)),"******",FLOOR(1000 + (RAND() * 9000)) ) as trade_card_no,
--     				FLOOR(#directAmount# + (FLOOR(RAND() * #indirectAmount#)*100)) as trade_amount,
    				DATE_FORMAT(CONCAT(DATE_SUB(curdate(),INTERVAL 1 DAY),' ',LPAD(FLOOR(0 + (RAND() * 23)),2,0),':',LPAD(FLOOR(0 + (RAND() * 59)),2,0),':',LPAD(FLOOR(0 + (RAND() * 59)),2,0)),'%Y-%m-%d %T') as trade_time,
    				REPLACE(UUID(),"-","")  as trade_no,
    				1 as trade_rate,
    				3 as brokerage_amount,
    				3 as source_type
    from order_info o_i
    left join (
    			select a.order_id as order_id,IFNULL(sum(b.trade_amount),0) as sumamount
    			from pos_info a
    			left join pos_trade_deal b on a.pos_sn = b.pos_sn   --出问题的条件
                        and b.trade_time > ADDDATE(CURRENT_DATE(),-1) 
                        and b.trade_time < CURRENT_DATE()
    			where a.status = 2 and a.order_id != 0 
    			GROUP BY a.order_id
    ) t_a on t_a.order_id = o_i.id
    left join pos_info p_i on p_i.order_id = o_i.id 
    where o_i.del = 0 and o_i.project_status = 3 
    	and t_a.sumamount = 0   and RAND()<(20/100)
    	and p_i.id is not null

```

 数据量为450   概率结果为 80-120之间跳动

分析结果：

```
id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
1	PRIMARY	o_i	ALL	PRIMARY	25	4	Using where
1	PRIMARY	p_i	ref	PRIMARY,idx_pos_opu_id	idx_pos_opu_id	8	chuangshoubao.o_i.id	12865	90	Using index condition
1	PRIMARY	<derived2>	ref	<auto_key0>	<auto_key0>	23	chuangshoubao.o_i.id,const	10	100	Using where; Using index
2	DERIVED	a	range	idx_pos_opu_id	idx_pos_opu_id	8	662	10	Using index condition; Using where
2	DERIVED	b	ref	idx_mer_code,idx_trade_time	idx_mer_code	96	chuangshoubao.a.merchant_code	10	100	Using where
```

**1	PRIMARY	<derived2>	ref	<auto_key0>	<auto_key0>	23	chuangshoubao.o_i.id,const	10	100	Using where; Using index**

### 分析处理

根据参考资料：<https://www.cnblogs.com/lYng/p/9441467.html>

这里的derived是叫做派生表的东东：解释为子查询–>在From后where前的子查询 。

> MySQL 5.7开始优化器引入derived_merge，可以理解为Oracle的子查询展开，有优化器参数optimizer_switch='derived_merge=ON’来控制，默认为打开。

>  但是仍然有很多限制，当派生子查询存在以下操作时该特性无法生效：UNION 、GROUP BY、DISTINCT、LIMIT/OFFSET以及聚合操作



这里对比之后，猜测是索引之类的导致有问题的SQL子查询，使用了全表查询。

最后在核对了索引之后，结果依旧。【索引没有问题】

**最后，发现是merchant_code这个字段被设置为is not null 导致的【这个是pos_sn字段唯一的区别了】**

去掉不为空的限制之后，就突然妥妥的没问题了(ノ｀Д)ノ  

完全不知道这中间发生了什么？.?

2019-03-13 小杭
