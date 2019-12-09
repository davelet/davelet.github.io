---
layout: post
title: Redis集群下的mget（Spring RedisTemplate版）
categories: [dev]
tags: [redis, java]
---

前面《[Redis的pipeline和mget](\redismget)》中说了，redis的批量操作命令对集群都不友好，因为Redis的官方集群方案是把key通过crc16计算hash映射到16384个桶上，落到哪个桶上就落到哪个机器上。网上的《[缓存无底洞问题](http://ifeve.com/redis-multiget-hole)》讲了几种在集群下使用mget的方案：一是拆成多次get，二是根据机器节点顺序查询，三是根据节点并行查询，四是使用hash tag。
接下来我们用Spring提供的RedisTemplate进行演示。你可以看到，RedisTemplate使用的就是第三种方案。

我本地起来3个节点构成一个集群（详细的搭建过程参考《[Mac 搭建 Redis 集群](https://www.jianshu.com/p/a32542ce4c0b)》），项目指定了地址是：
```bash
spring.redis.cluster.nodes=127.0.0.1:17000,127.0.0.1:17001,127.0.0.1:17002
```

接下来我们直接使用默认方式插入100条数据并获取：
```java
var map = new HashMap<String, String>(100, 1);
for (var i = 1; i <= 100; i++) {
    map.put("cluster_" + i, "c" + i);
}
var start = System.currentTimeMillis();
redisTemplate.opsForValue().multiSet(map);
var medium = System.currentTimeMillis();
List<String> strings = redisTemplate.opsForValue().multiGet(map.keySet());
var end = System.currentTimeMillis();
System.out.println(" \n " + (medium - start) + "\n" + (end - medium));
```

输出是：
```bash
397
17
```

---

---

接下来使用hash tag进行对比：
```java
for (var i = 1; i <= 100; i++) {
    map.put("{cluster}_" + i, "c" + i);
}
start = System.currentTimeMillis();
redisTemplate.opsForValue().multiSet(map);
medium = System.currentTimeMillis();
redisTemplate.opsForValue().multiGet(map.keySet());
end = System.currentTimeMillis();
System.out.println(" \n " + (medium - start) + "\n" + (end - medium));
```

hash tag就是在key中将一部分字符用大括号包起来做为tag，redis会只把tag进行hash做为key的hash。这样所有的hash都一样，所以会落到同一个节点上。对比代码的输出是：
```
3
3
```
可见在插入和查询上效率都提升了很多。

实际上如果是我们自己实现hash tag的查询效率可以更高。因为spring在查询前会对比所有tag的hash是否一致（实际是对比key的hash），当出现不一致时中断对比并走第三种方案：并行查询。否则要对比完所有的key后确定查询在单个节点上。这个对比过程对于我们来说是多余的，因为我们的目的就是让他们在同一个节点。

> Spring的这个逻辑可以参考类：org.springframework.data.redis.connection.jedis.JedisClusterStringCommands#mGet

完整的代码在：[https://github.com/davelet/redis-lua-101/tree/master/redis-cluster-multi](https://github.com/davelet/redis-lua-101/tree/master/redis-cluster-multi)
