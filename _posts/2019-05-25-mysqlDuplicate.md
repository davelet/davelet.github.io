---
layout: post
title: mysql的“UPSERT”
# subtitle: '或许是最漂亮的Jekyll主题'
date: 2019-05-25
categories: [dev]
tags: [mysql, database]
---
我最早接触到的数据库的upsert功能是MongoDB提供的，upsert能避免竞态问题。没有upsert的时候，需要先去数据库中查找符合条件的文档，然后再根据更新信息更新数据，这个在多线程或者多进程的情况下会产生资源竞争，使用upsert可以很好的避免这种情况的发生。Mysql没有提过（我没有见过）upsert这个概念，不过提供了语法实现类似的功能。

MongoDB的upsert是直接使用update方法，当被update的数据不存在的时候就插入一条。MySQL不一样，它是在insert的时候判断是否有唯一性约束校验不通过，不通过的时候并不是把数据整个替换过去，而是需要指定要替换的字段和要替换的值。如果不指定某个字段，这个字段就不会被更新；如果不指定这个字段要更新成的值也不允许，并不会自动更新为传进来的值。

语法如下：
```
INSERT INTO table(fields...) VALUES (values...)
  ON DUPLICATE KEY UPDATE feild = {value}
```

> PostgreSQL 9.5开始也提供了类似的功能，语法类似：INSERT INTO table VALUES (values...) ON CONFLICT (uni_idx) DO UPDATE SET feild = {value}

## upsert单条记录
创建如下表用作测试：
```
CREATE TABLE `upsert` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `value` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

（coming soon）
## 批量upsert

## 和replase的区别