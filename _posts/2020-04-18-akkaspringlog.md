---
layout: post
title: 简单监控akka在spring boot项目中的actor信箱
categories: [dev]
tags: [java, akka]
---

做为项目中的新鲜玩意，akka的性能是上线后特别要关注的地方。没有人对这东西很懂，所以不盯紧一点，随时可能演变成大事故 —— 而且是不知道怎么修复的那种事故。

因为我在项目中使用的actor都是单例，也就是所有的请求都会被actor以单线程处理，所以监控actor的信箱大小是很重要，万一业务量使得单线程actor处理不过来怎么办。

> 既然使用单例actor，为何不使用单线程的线程池呢？从运行机制上看，actor和线程池差别不大，如果没有特别的理由选谁就看爱好了。

除了信箱的大小，如果能监控actor的数量、占用的内存、占用计算资源等方面也是极好的。我在网上找了一阵，针对akka的监控有几个，主要是

 - Lightbend Telemetry： https://developer.lightbend.com/docs/telemetry/current/home.html
 - Kamon: https://kamon.io/docs/v1/instrumentation/akka/metrics/
 - Github gist: https://gist.github.com/patriknw/5946678

前两个是系统的监控，能力强大，监控方面也多。第一个需要注册，我就放弃了；第二个搞了半天，愣是跑不起来。第三个看起来还行。这里就说一下第三个的用法。

```java 
package akka.contrib.mailbox;

import com.typesafe.config.Config;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicInteger;
import scala.Option;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.dispatch.Envelope;
import akka.dispatch.MailboxType;
import akka.dispatch.MessageQueue;
import akka.dispatch.ProducesMessageQueue;
import akka.dispatch.UnboundedMailbox;
import akka.event.Logging;
import akka.event.LoggingAdapter;

/**
 * Logs the mailbox size when exceeding the configured limit. It logs at most
 * once per second when the messages are enqueued or dequeued.
 *
 * Configuration:
 *
 * <pre>
 * akka.actor.default-mailbox {
 *   mailbox-type = akka.contrib.mailbox.LoggingMailboxType
 *   size-limit = 20
 * }
 * </pre>
 */
public class LoggingMailboxType implements MailboxType, ProducesMessageQueue<UnboundedMailbox.MessageQueue> {
  private final Config config;

  public LoggingMailboxType(ActorSystem.Settings settings, Config config) {
    this.config = config;
  }

  @Override
  public MessageQueue create(Option<ActorRef> owner, Option<ActorSystem> system) {
    if (owner.isEmpty() || system.isEmpty())
      throw new IllegalArgumentException("no mailbox owner or system given");
    int sizeLimit = config.getInt("size-limit");
    return new LoggingMailbox(owner.get(), system.get(), sizeLimit);
  }

  static class LoggingMailbox implements MessageQueue {

    private final Queue<Envelope> queue = new ConcurrentLinkedQueue<Envelope>();

    private final int sizeLimit;
    private final LoggingAdapter log;
    private final long interval = 1000000000L; // 1 s, in nanoseconds
    private final String path;
    volatile private long logTime = System.nanoTime();
    private final AtomicInteger queueSize = new AtomicInteger();
    private final AtomicInteger dequeueCount = new AtomicInteger();

    LoggingMailbox(ActorRef owner, ActorSystem system, int sizeLimit) {
      this.path = owner.path().toString();
      this.sizeLimit = sizeLimit;
      this.log = Logging.getLogger(system, LoggingMailbox.class);
    }

    @Override
    public Envelope dequeue() {
      Envelope x = queue.poll();
      if (x != null) {
        int size = queueSize.decrementAndGet();
        dequeueCount.incrementAndGet();
        logSize(size);
      }
      return x;
    }

    @Override
    public void enqueue(ActorRef receiver, Envelope handle) {
      queue.offer(handle);
      int size = queueSize.incrementAndGet();
      logSize(size);
    }

    private void logSize(int size) {
      if (size >= sizeLimit) {
        long now = System.nanoTime();
        if (now - logTime > interval) {
          double msgPerSecond = ((double) dequeueCount.get()) / (((double) (now - logTime)) / 1000000000L);
          logTime = now;
          dequeueCount.set(0);
          log.info("Mailbox size for [{}] is [{}], processing [{}] msg/s", path, size,
              String.format("%2.2f", msgPerSecond));
        }
      }
    }

    @Override
    public int numberOfMessages() {
      return queueSize.get();
    }

    @Override
    public boolean hasMessages() {
      return !queue.isEmpty();
    }

    @Override
    public void cleanUp(ActorRef owner, MessageQueue deadLetters) {
      for (Envelope handle : queue) {
        deadLetters.enqueue(owner, handle);
      }
    }
  }
}
```

# 使用步骤

## 1. 复制文件
把整个文件复制到项目的合适包中，假设包名是a.b.c。

## 2. 创建application.conf
在主模块的src/main/resources下面创建文件application.conf，内容：
```conf
akka {
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel = "INFO"
  logging-filter = "akka.event.slf4j.Slf4jLoggingFilter"
  log-config-on-start = false
  logger-startup-timeout = 20s
  actor {
    default-mailbox {
      mailbox-type = a.b.c.LoggingMailboxType
      size-limit = 10
    }
    debug {
      lifecycle = false
    }
  }
  log-dead-letters = 10
  log-dead-letters-during-shutdown = on
}
```
这里配置了日志类型、邮箱类型（使用a.b.c.LoggingMailboxType）等。你也可以使用最简单的配置：
```conf
akka.actor.default-mailbox {
    mailbox-type = a.b.c.LoggingMailboxType
    size-limit = 10
}
```
配置中的size-limit会在代码中获取到，当大小超过它的时候会打印日志。

## 修复父类
如果到上面一步就开动项目，我这边一直报日志系统连接超时。这个错误让我十分纳闷：日志有什么超时的？后来引入了包
```xml
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-slf4j_2.12</artifactId>
    <version>2.5.26</version>
</dependency>
```
启动才得到报错：需要使用UnboundedMessageQueueSemantics的实现类才行。

于是将上面代码中的
```java 
  static class LoggingMailbox implements MessageQueue {
```
改成
```java 
  static class LoggingMailbox extends UnboundedMailbox.MessageQueue {
```
才启动成功。

暂且这样吧！