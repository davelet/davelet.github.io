---
layout: post
title: Java 的 elasticsearch Rollover API 简介
categories: [dev]
tags: [java, elasticsearch]
---
有时候我们需要用ES保存海量的流水数据，比如日志、比如轨迹等等。这种数据时效性低，几个月前（甚至一个月前）的数据价值可能就没有了，可做删除或归档处理。ES 提供了rollover机制自动分隔索引（类似于业务log可根据日期和大小分隔），并提供了shrink机制归档过期索引。本篇简单介绍一下rollover api的使用。

# 别名
es给索引提供了别名。索引别名有两种，一种是读索引，可以包含多个索引；一种是写索引，只能指向一个索引。

> 除了索引别名，还有字段别名。这里不讨论

为了方便的使用rollover api，我们需要结合别名。

# 模板
模板和其他软件的功能一样，都是简化创建过程的。模板需要指定一个模式，索引在创建的时候会先去查询模板，看自己的名字符合哪些模板就套用。索引在rollover的时候一般索引前缀都一样，何况结构完全一样，所以我们可以使用模板。更重要的是，我们可以指定符合模板的索引可以被包含到哪个别名中（这里只能是读索引）。

比如我们的一系列索引都以`my-index`开头，后面跟流水号。那模板的模式就可以指定为`my-index-*`：
```
PutIndexTemplateRequest pitr = new PutIndexTemplateRequest()
                .name(yourTemplateName) // 指定模板的名字，就和索引名一样
                .patterns(Collections.singletonList("my-index-*")) // 模式
                .alias(new Alias(YourReadAlias)); // 匹配模板的索引具有的别名
transportClient.admin().indices().putTemplate(pitr);
```
创建好模板以后就可以创建索引了，第一个索引一般叫`my-index-1`，而且要指定写别名：
```
CreateIndexRequest cir = new CreateIndexRequest()
        .index("my-index-1")
        .alias(new Alias(yourWriteAlias));
transportClient.admin().indices().create(cir);
```
# 滚动
es的rollover不是自动的，必须调用相应的api触发滚动；当然不符合滚动条件不会真正滚动的。
```
RolloverRequest rr = new RolloverRequest(yourWriteAlias, null);
rr.addMaxIndexAgeCondition(new TimeValue(10, TimeUnit.DAYS));
rr.addMaxIndexDocsCondition(1_0000_0000);
ActionFuture<RolloverResponse> index = transportClient.admin().indices().rolloversIndex(rr);
```
滚动请求必须传入一个写别名，和第一个索引创建的时候是同一个。
滚动可以触发的条件有三种，都是`org.elasticsearch.action.admin.indices.rollover.Condition`的子类。

> 滚动条件是深度编写到es源码中的，所以我们没法自己实现自定义条件

- MaxAgeCondition，索引创建满一定时间就滚动
- MaxDocsCondition，索引内文档量超过阈值就滚动
- MaxSizeCondition，索引占用空间超过阈值就滚动

上面用到了前两个条件：满10天或一亿条。

> 示例源码库：[https://github.com/davelet/es-rollover-api-101](https://github.com/davelet/es-rollover-api-101)

---
建议：

> 大量数据的持续上报会极大占用ES资源。
为提升保存速度，建议使用es bulk api进行批量插入，可高性能的单次插入成千上万条文档。
因此插入es前将记录可以先缓存到redis中，当缓存量足够大时再批量插入es。还可以给对象加时间戳，每次插入对比一下超过一定时间就不再等待数量足够。
redis list没有提供原子性的批量pop功能，需要先range再delete，所以插入到es后要将新的记录写入新的redis list中（可以通过key轮盘实现）。
这种设计带来的不足是记录有延迟，尤其在低峰期，可能要一直到下一条上报才能把很久前的写入es。