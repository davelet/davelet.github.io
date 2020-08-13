---
layout: post
title: 项目中逻辑执行异常后的重试机制
categories: [dev]
tags: [java]
---

已经有10来天没有写博客了。不是因为没有东西写——其实想写的东西还挺多的——是因为git pages被墙的原因，“生殖隔离”导致我没有心情来写。

> 不说github是最大的同性交友社区吗

在我们日常作业中，经常会有依赖方接口不通、本地数据有误等情况，导致我们的业务逻辑暂时没法继续走下去。通常我们都是报个错（程序异常了，当然会抛出来），最多捕获一下多打个日志：这种做法是所有异常场景能普适的行为。但是针对一些临时性的环境异常，比如对方在发版，可能稍等一会再次重试就可以了。再比如说，由于数据有问题，出现异常后数据维护的人员通过告警快速修复了数据；程序如果能自动重试就很有可能可以将业务逻辑走完。

所以这里简单提供两种重试机制。一种是基于自定义的延迟消费策略，一种是基于Spring提供的重试机制。

# 一、基于akka延迟信件的机制

> 这种机制完全可以使用mq代替。发现异常以后重试一次就好了，只不过由于要设置延迟，所以可以通过Mq走异步定时消费

使用akka是因为我的项目中广泛使用了akka actor（关于akka的用法可以参考我前面的文章）。由于我的逻辑都是起源于akka的信件，所以在处理信件的时候如果异常了，那只要定时给自己发一封同样的信再次触发逻辑处理即可。

首先定义一个可以重试的事件类型：
```java 
@ToString(callSuper = true)
public abstract class AbstractRetryableEvent extends AbstractEvent implements Retryable{
    private boolean retryable = true;//处理当前事件时发生异常是否重试

    @Override
    public boolean retryable() {
        return retryable;
    }

    @Override
    public void reset() {
        retryable = true;
    }

    @Override
    public void revert() {
        retryable = !retryable;
    }
}
```
然后在接收到事件后进行可重试判断。下面是只重试一次的逻辑：
```java 
try {
    // 业务逻辑处理
} catch (Exception e) {
    if (event.retryable()) {//发生异常 并且可以重试
        event.revert();// 只重试一次，所以标记为不可重试
        getContext().system().scheduler().scheduleOnce(Duration.ofMinutes(2), getSelf(), event, getContext().system().dispatcher(), getSelf());// 发送一个延迟2分钟的信件
    } else {
      // 执行时发生了异常，并且当前不可重试，走告警逻辑
        ActorRef logActor = SpringUtils.getBean("logActor");
        logActor.tell(new LogEvent(xxx), getSelf());
    }
}
```
对于需要重试多次的可以加入一个自增变量代表当前重试次数，也可以使用一个时间变量标志最后重试时间。比如
```java 
if (event.retryable()) {
  if (count == 0) {// 数据依然不满足要求，需要进行重试
      LocalDateTime now = LocalDateTime.now();
      if (Duration.between(event.getSourceTime(), now).getSeconds() < TimeUnit.MINUTES.toSeconds(10)) {
          // 超过时限就不再重试
          event.reset();//需要重试
      } else {
          event.revert();//超时不再重试
      }
  } else {// 数据已经满足要求，可以继续业务逻辑
      event.revert();//标记为不再重试
      ActorRef syncActor = SpringUtils.getBean("businessActor");
      syncActor.tell(new BusinessLogicEvent(), getSelf());//继续业务逻辑
  }
  getContext().system().scheduler().scheduleOnce(Duration.ofMinutes(param.getStep()), getSelf(), event, getContext().system().dispatcher(), getSelf());
} else {
  getContext().stop(getSelf());// 可选的销毁actor
}
```
通过注释也能很清晰的了解过程。

# 二、Spring 的重试注解
既然我们能遇到需要重试的场景，其他程序员也早遇到了。所以完善的Spring框架也提供了重试机制。下面以springBoot项目为例。
使用spring retry需要先引入依赖：
```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
    <version>1.2.2.RELEASE</version>
</dependency>
```

首先要在配置类上启动重试机制：
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.retry.annotation.EnableRetry;

@EnableRetry
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
```

在需要重试的方法上增加重试配置注解：
```java
    @Retryable(
      value = {MyBusinessException.class},
      maxAttempts = 1,
      backoff = @Backoff(
        delay = 1000L
      )
   )
   public String getString() {
     // 业务逻辑，抛异常需要重试
   }
```
@Retryable 注解的属性很多，上面三个字段分别指的是在那种异常下需要重试（可以是运行时异常或方法签名上的throw异常，会拦截该类及其子类，不匹配的异常类型直接报错），最大重试次数（默认3次，达到重试次数依然异常就会报错了，除非有降级方法。第一次就失败也是包含在的，第三次还失败就报错），间隔多久重试（默认1秒）。其他

如果在重试超过了次数后希望降低，可以在同一个类里面增加一个Recover方法：
```java
    @Recover
    public String r() {
        return "default";
    }
```
这个Recover方法的返回值必须是Retryable方法相同或其子类。另外这个方法可以接收一个异常类型的参数，表示Retryable方法抛出哪种异常可以走哪个降级方法，因为Retryable方法的value值是一个集合。
如果一个类里面有多个Retryable方法，并且通过上面这个过程能够匹配到多个降级方法，那spring会根据jvm的逻辑寻找“距离最近”的方法，具体过程可以看 RecoverAnnotationRecoveryHandler 这个类。如果多个Retryable方法匹配到同一个Recover方法，它们都会使用这个Recover方法的返回值。

