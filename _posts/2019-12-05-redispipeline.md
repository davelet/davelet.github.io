---
layout: post
title: Redis的pipeline技术(java版)
categories: [dev]
tags: [redis, java]
---

redis的pipeline提供的是一种批量提交请求的能力。因为Redis本身速度很快，操作都在内存里；而操作却需要通过网络，网络再好速度也慢。如果需要大批量操作，网络上的时间十分可观。pipeline就是将请求有序的组织在一起，一次发给服务器顺序执行。

先来看一下使用Pipeline和不使用的对比：
```
private static final int COMMAND_NUM = 100000;

public static void main(String[] args) {
    Jedis jedis = new Jedis("localhost", 6379);
    withoutPipeline(jedis);
    withPipeline(jedis);
    withoutPipeline(jedis);// 再来一次
    withPipeline(jedis);
}

private static void withoutPipeline(Jedis jedis) {
    var start = Instant.now();
    for (var i = 0; i < COMMAND_NUM; i++) {
        jedis.set("no_pipe_" + i, String.valueOf(i), SetParams.setParams().ex(60));
    }
    var end = Instant.now();
    var cost = Duration.between(start, end);
    System.out.println("withoutPipeline cost : " + cost);
}

private static void withPipeline(Jedis jedis) {
    var pipe = jedis.pipelined();
    var start_pipe = Instant.now();
    for (var i = 0; i < COMMAND_NUM; i++) {
        pipe.set("pipe_" + i, String.valueOf(i), SetParams.setParams().ex(60));
    }
    pipe.sync(); // 获取所有的response
    var end_pipe = Instant.now();
    var cost_pipe = Duration.between(start_pipe, end_pipe);
    System.out.println("withPipeline cost : " + cost_pipe);
}
```
代码很简单，就是分别打印执行10万次set的时间。这段代码在我电脑上的redis环境执行输出如下：
```
withoutPipeline cost : PT3.696743S
withPipeline cost : PT0.280996S
withoutPipeline cost : PT3.038304S
withPipeline cost : PT0.23413S
```
为了公平和预热，我这里穿插执行了两次。可以看到不用管道需要3秒多，使用管道是200多不到300毫秒。差异明显啊！

---

也可以在Lua脚本中执行多次set：
```
for i=1,tonumber(KEYS[1]) do
    redis.call('set', 'lua_'..i, '1')
end
return 'done'
```

然后执行10万次：
```java
var list = new ArrayList<String>(1);
list.add("100000");

StringRedisSerializer utf8 = StringRedisSerializer.UTF_8;
var start = Instant.now();
var script = new DefaultRedisScript<String>();
script.setResultType(String.class);
script.setLocation(new ClassPathResource("lua.lua"));
String execute = redisTemplate.execute(script, utf8, utf8, list);
System.out.println(execute + " " + Duration.between(start, Instant.now()));
start = Instant.now();
execute = redisTemplate.execute(script, utf8, utf8, list);
System.out.println(execute + " " + Duration.between(start, Instant.now()));
start = Instant.now();
execute = redisTemplate.execute(script, utf8, utf8, list);
System.out.println(execute + " " + Duration.between(start, Instant.now()));
```
打印结果是：
```
done PT0.432738S
done PT0.194804S
done PT0.217526S
```
稳定后时间在200毫秒左右，略优于直接Pipeline。

---

完整代码：[https://github.com/davelet/redis-lua-101](https://github.com/davelet/redis-lua-101)