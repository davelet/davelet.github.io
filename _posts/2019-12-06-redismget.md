---
layout: post
title: Redis的pipeline和mget
categories: [dev]
tags: [redis, java]
---

前面《[Redis的pipeline技术](\redispipeline)》介绍了redis的pipeline技术。pipeline会将请求合在一起，一次发给服务器执行。另外，Redis还提供了mset和mget命令，用于一次性操作多个key。这里我们看一下它们的对比。

> 注意：redis的批量操作命令对集群都不友好，如果想要使用需要深入调研和针对性开发

# pipeline get
首先看管道获取：
```java
var start = Instant.now();
redisTemplate.executePipelined((RedisCallback<String>) redisConnection -> {
    for (var i = 1; i <= 10_000; i++) {
        redisConnection.get(StringRedisSerializer.UTF_8.serialize("lua_" + i));
    }
    return null;
});
System.out.println("pipe: " + Duration.between(start, Instant.now()));
```
这段代码会获取一万个之前插入的key，经过多次尝试，我本地耗时在500毫秒左右。

> 注意execute里面的方法只能返回null， 不然会报错。

# mget
现在看一下mget：
```java
var start = Instant.now();
List<String> list = new ArrayList<>(10_000);
for (var i = 1; i <= 10_000; i++) {
    list.add("lua_" + i);
}

redisTemplate.opsForValue().multiGet(list);
System.out.println("mget: " + Duration.between(start, Instant.now()));
```

这段代码多次执行的平均时间在30毫秒左右。也就是比pipeline快了一个量级以上。

---


# mset
在前面的文章我们对比了使用pipeline和不使用的区别，现在我们加入mset看一下。

> 注意：刚才的对比使用的是spring的redisTemplate，接下来我们使用Jedis。同样的操作redisTemplate要比直接使用jedis慢不少,所以它们之间没有对比的意义

```java
private static void withMset(Jedis jedis) {
    var start_pipe = Instant.now();
    Map<byte[], byte[]> src = new HashMap<>(10_0000, 1);
    for (var i = 0; i < COMMAND_NUM; i++) {
        src.put(("pipe_" + i).getBytes(), "3".getBytes());
    }
    byte[][] bs = toByteArrays(src);
    jedis.mset(bs);

    var end_pipe = Instant.now();
    var cost_pipe = Duration.between(start_pipe, end_pipe);
    System.out.println("withMset cost : " + cost_pipe);
}

private static byte[][] toByteArrays(Map<byte[], byte[]> source) {
    byte[][] result = new byte[source.size() * 2][];
    int index = 0;

    Map.Entry entry;
    for(Iterator var3 = source.entrySet().iterator(); var3.hasNext(); result[index++] = (byte[])entry.getValue()) {
        entry = (Map.Entry)var3.next();
        result[index++] = (byte[])entry.getKey();
    }

    return result;
}
```

输出结果类似于：
```bash
withoutPipeline cost : PT3.238862S
withPipeline cost : PT0.216674S
withMset cost : PT0.191673S
```
也就是说mset要略优于pipeline。
---

# 对比set、pipe、mset、lua
现在我们把所有四种设置方法：直接多次set、使用管道、使用mset和使用脚本一起进行比较，
完整代码：[https://github.com/davelet/redis-lua-101/blob/master/redis-pipeline-jedis/src/main/java/info/manxi/redis/pipeline/JedisTest.java](https://github.com/davelet/redis-lua-101/blob/master/redis-pipeline-jedis/src/main/java/info/manxi/redis/pipeline/JedisTest.java)。

对比结果大致是：
```
withoutPipeline cost : PT3.049489S
withPipeline cost : PT0.254727S
withMset cost : PT0.190823S
withlua cost : PT0.218129S
```

mset和luau脚本相仿，略快于pipeline。