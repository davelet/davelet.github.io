---
layout: post
title: MySQL 的 “UPSERT”
date: 2019-05-25
categories: [dev]
tags: [mysql, database]
---
我最早接触到的数据库的upsert功能是MongoDB提供的，upsert能避免竞态问题。没有upsert的时候，需要先去数据库中查找符合条件的文档，然后再根据更新信息更新数据，这个在多线程或者多进程的情况下会产生资源竞争，使用upsert可以很好的避免这种情况的发生。Mysql没有提过（我没有见过）upsert这个概念，不过提供了语法实现类似的功能。

MongoDB的upsert是直接使用update方法，当被update的数据不存在的时候就插入一条。MySQL不一样，它是在insert的时候判断是否有唯一性约束校验不通过(包括主键约束和唯一性索引)，不通过的时候并不是把数据整个替换过去，而是需要指定要替换的字段和要替换的值。如果不指定某个字段，这个字段就不会被更新；如果不指定这个字段要更新成的值也不允许，并不会自动更新为传进来的值。

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
我们先验证主键约束：
```
insert into upsert values (1, "1");
insert into upsert values (1, "1");
```
很明显，两次插入相同id的记录，第二次会失败：
```
Duplicate entry '1' for key 'PRIMARY'
Error code 1062.
```
把第二次的语句改成
```
insert into upsert values (1, "1") on duplicate key update value = "11";
```
更新成功：
```
Total Rows Affected: 2
Execution Time: O secs
```
虽然成功了，但是返回值可能让人纳闷：为啥是影响了2行呢，不是只更新了一行吗？

为了解答这个问题，我们继续执行一下刚才的语句，得到：
```
Total Rows Affected: 0
Execution Time: O secs
```
咦，为啥又是0条了呢？难道没匹配吗？不可能！难道没更新吗？是的。

“insert on duplicate key update”在没有冲突且插入成功后返回1，如果主键冲突，并且成功更新了某个字段，返回2；如果冲突了，但是要更新的字段已经是期望值，则不再更新，返回0。
> 第三种情形和update一样。当update要设置的值和数据库本来值一样时也不会再更新

接下来验证唯一索引，给value字段加上索引：
```
create unique index uni on upsert (value);
insert upsert values (2, "11");
```
得到
```
Duplicate entry '11' for key 'uni'
Error code 1062.
```
因为value是11的已经存在了。将语句改成
```
insert upsert values (2, "11") on duplicate key update  value="2"
```
成功返回2条受影响。
## 批量upsert
更新一条的可以。但是更新多条怎么办？比如我本来的语句是
```
insert into upsert values (1,"a"),(2,"b")
```
由于期望在冲突时upsert所以想加上on duplicate key update，可是update后面咋写呢？如果写成
```
insert into upsert values (1,"a"),(2,"b") on duplicate key update value = "x"
```
则所有的冲突条目的value都会是x，但是我想要每一条都变成我传入的值怎么办？MySQL提供了values(col_name)函数（注意不是insert values那个关键字，当然写法一样）获取每一条的输入值：
```
insert into upsert values (1,"a"),(2,"b") on duplicate key update value = values(value)
```
意思就是在冲突的记录上，将字段value改成传入的字段value的值（不冲突的正常插入）。

> values()函数只能用在这个场景，其他地方不支持

另外，插入单条也可以使用values函数，你可以试试。
## 和replace的区别

有人说，这个看起来和replase语法类似啊。

```
REPLACE into upsert values (1,"x"),(2,"y"),(3,"z")
```
会把本来存在的前两条先删掉，再把三条重新插入，返回5。有什么问题吗？
```
REPLACE into upsert(value) values ("x"),("y"),("z")
```
因为主键是自增的，所以我们不再设置主键值。

上面的x,y,z已经在数据库存在，所以会先删掉，再插入3条新的。新插入的3条id就不再是1,2,3了。

