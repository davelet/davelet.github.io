---
layout: post
title: akka Actor入门
categories: [dev]
tags: [akka, actor]
---

Akka 组件不少，最基础的就是akka-actor模块，这是所有其他模块的基础。没有这个模块，其他就没法工作。而这个模块呢，却可以独立工作。之前发过几篇akka相关的文章了（见[akka标签](/tags/#akka)），不过这里还是再继续稍微讲一下actor的特性。

actor的基本原理可以看一下[akka使用场景及原理简介](/akka-intro/)。Akka的基本分发单元就是一个个的actor，当然了akka就是根据actor思想创建的一套框架。由于是基于Actor的，akka也具备了灵活性和方便的扩展性。每个Actor都是单线程工作的，所以我们使用Actor就不用编写同步代码了：而且也不能编写！

> 记住：在actor中不能使用volatile和synchronized关键字

actor是消息驱动的，没有消息的时候它们就闲着，有消息发给它们了，它们就从线程池中取一个线程开始执行，线程是事件分发器分配的。每个分发器都有一个线程池伴随。actor执行完业务逻辑就把线程归还到线程池。

每个actor都是独立的计算单元。它们有如下特征：
 - actor是有状态的，并且包含了部分业务逻辑
 - actor间的交互只能通过消息，决不可能通过方法调用
 - actor都有自己唯一的地址（可以通过ActorRef的path()方法获取）和信箱
 - actor是按照顺序处理信箱中的消息的（默认FIFO）
 - actor系统的组织结构是树形的
 - actor可以创建子actor也可以销毁它们

### 状态
actor通常会携带变量以表征其状态（可能是状态机、计数器、回调监听、等待中的请求等），这些变量不能别其他actor破坏，actor机制保证了这一点。所以你可以随便写代码，完全不用考虑会不会线程不安全。

### 行为
每条消息发送出来都对应了一种actor的行为，行为就是actor对收到的消息做出的反应。例如，如果客户端被授权，则转发请求，否则拒绝该请求。行为可能随着时间改变。

### 信箱
消息的处理者是actor，发送者是其他actor（或外部系统），连接二者的就是接收者的信箱。每个actor都有一个信箱（有且仅有），信箱是一个队列，按时间入队。同一个actor发出的消息对相同的actor是有序的。

下面是一个actor的例子：
```
class SummingActor extends Actor {
    // 状态
    var sum = 0
    // 行为
    override def receive: Receive = {
        // 如果是整数
        case x: Int => sum = sum + x
            println(s”my state as sum is $sum”)
        // 其他类型消息
        case _ => println(“I don’t know what are you talking about”)
    }
}
```
这个actor里面就定义了状态和行为。

# actor 层级
akka系统又两层actor：最顶层是根守卫（root guardian），它由两个孩子，分别是/user和/system。我们自己创建的actor都是/user的子actor。假设我们的actorSystem叫“mySystem”，创建的actor叫“myFirstActor”，那么这个actor的路径就是akka://mySystem/user/myFirstActor。
> ActorSystem system = ActorSystem.create("mySystem");
> ActorRef actor = system.actorOf(..., "myFirstActor")；

<div align="center">
<img width="80%" src="/images/post/actor_hi.png">
</div>
