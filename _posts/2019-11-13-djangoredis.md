---
layout: post
title: 在Django项目中使用Redis
categories: [dev]
tags: [python, redis]
---

在Django项目中使用Redis其实就是Python项目中使用Redis，和是否Django环境无关。这篇文章说一下如何集成Redis集群。如果是本地或者其他非集群环境，可能不适合这种方法，而是有更简单的方法。这里不说了就。

首先当然还是下载依赖：
```
pip install redis-py-cluster
```
从名字就可以看出，是针对Redis集群的。

---

假设集群地址是
```
redis_str = 'redis-node1-t1.yh.test:7001,redis-node1-t1.yh.test:7002,redis-node2-t1.yh.test:7001,redis-node2-t1.yh.test:7002,redis-node3-t1.yh.test:7001,redis-node3-t1.yh.test:7002'
```

把它们分解，创建集群节点列表：
```
redis_strs = redis_str.split(',')

startup_nodes = []
for up in redis_strs:
    uandp = up.split(':')
    startup_nodes.append({'host': uandp[0], 'port': uandp[1]})
```

创建客户端连接很简单：
```
from rediscluster import RedisCluster

redis_client = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)
```
后面的参数decode_responses如果没指定为True，在python3下返回内容会是二进制。

---

后面的操作直接使用redis_client即可，比如：
```
redis_client.set('foo', 'bar')

redis_client.expire('foo', 10)

redis_client.setnx('foo', '1')

redis_client.delete('foo')
```