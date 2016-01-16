---
layout: post
title: "sql or keyword optimize"
date: 2014-08-31 19:48:56 +0800
comments: true
categories: sql
tags: sql
share: true
description: 一次sql or 关键字调优
toc: true
---
碰见一个sql执行慢的问题，优化后执行就比较快了，记录一下优化过程。

<!--more-->

---

## 背景介绍

原始部分sql如下 <font color="#7c837f">(其中'2014-08-20 11:20:00'是变量传入进去的)</font>

``` sql
  SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table1 WHERE
CreateTime >= '2014-08-20 11:20:00' AND
CreateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
OR DeleteTime >= '2014-08-20 11:20:00' AND
DeleteTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
OR UpdateTime >= '2014-08-20 11:20:00' AND
UpdateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
  UNION ALL
  SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table2 WHERE
CreateTime >= '2014-08-20 11:20:00' AND
CreateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
OR DeleteTime >= '2014-08-20 11:20:00' AND
DeleteTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
OR UpdateTime >= '2014-08-20 11:20:00' AND
UpdateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
  UNION ALL

-- 等等直到table7
```

当时sql执行慢，因此直接在<font color="green">CreateTime、DeleteTime、UpdateTime </font>这3个字段建索引。

效果还不错，其中慢的30个DB，好了28个。但是，还有2个 mysql 实例 仍旧慢。而且这2个Mysql实例是在一个物理机上的不同端口。

30分钟轮转一次，先A慢B快，后B慢A快，好神奇。想着是不是有什么计划任务在机器上跑，每30分钟一次？

了解到 由于是备库，因此机器差，加上主备同步的因素，因此慢了。而且还有个重要因素，看了一下sql的执行计划，发现没有走索引...为什么？

``` sql
id	select_type	table	 type	possible_keys	  key	key_len	ref	rows	Extra
1	  PRIMARY     table1	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	83343	Using where
2	  UNION	      table2	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	82370	Using where
3	  UNION	      table3	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	82351	Using where
4	  UNION	      table4	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	83734	Using where
5	  UNION	      table5	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	82726	Using where
6	  UNION	      table6	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	82437	Using where
7	  UNION	      table7	ALL	index_CreateTime,index_DeleteTime,index_UpdateTime	81716	Using where
	UNION RESULT	<union1,2,3,4,5,6,7>	ALL
```

是OR 关键字导致没走索引。因此把or 改成union，类似这样的sql

``` sql
SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table1 WHERE
CreateTime >= '2014-08-20 11:20:00' AND
CreateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
union
SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table1 WHERE
DeleteTime >= '2014-08-20 11:20:00' AND
DeleteTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
union
SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table1 WHERE
UpdateTime >= '2014-08-20 11:20:00' AND
UpdateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)

union

SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table2 WHERE
CreateTime >= '2014-08-20 11:20:00' AND
CreateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
union
SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table2 WHERE
DeleteTime >= '2014-08-20 11:20:00' AND
DeleteTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)
union
SELECT '2014-08-20 11:20:00' as DataDate, A ,B, C,CreateTime,DeleteTime,UpdateTime FROM db.table2 WHERE
UpdateTime >= '2014-08-20 11:20:00' AND
UpdateTime < DATE_ADD('2014-08-20 11:20:00',INTERVAL 20 MINUTE)

-- 只拿2张表示例一下，实际上有7张
```

只写了2个表，为了查看一下执行计划，走了范围索引，搞定了。

``` sql
id	select_type	table	type	possible_keys	key	key_len	ref	rows	Extra
1	PRIMARY	  table1	range	index_CreateTime	index_CreateTime	9		1	Using where
2	UNION	  table1	range	index_DeleteTime	index_DeleteTime	9		1	Using where
3	UNION	  table1	range	index_UpdateTime	index_UpdateTime	9		1	Using where
4	UNION	  table2	range	index_CreateTime	index_CreateTime	9		1	Using where
5	UNION	  table2	range	index_DeleteTime	index_DeleteTime	9		2	Using where
6	UNION	  table2	range	index_UpdateTime	index_UpdateTime	9		3	Using where
	UNION RESULT	<union1,2,3,4,5,6>	ALL
```
