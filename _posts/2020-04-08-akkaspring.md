---
layout: post
title: akka在spring boot项目中的使用
categories: [dev]
tags: [java, akka]
---

学习akka一年多了，用起来还是挺方便。最近的项目中用的逐渐多了，这里简单记录一下。

> 项目是spring boot上的，所以主要说一下与spring的整合

# 应用场景的确定
什么时候应该使用同步，什么使用应该使用异步？选择好适用场景才能更好的应用技术。akka是高并发环境下线程安全的框架，所以要确定何种场景可以使用akka。

我的项目中需要和其他系统对接，其他系统发数据过来，我处理后回调过去。因为不是同步请求，所以可以使用akka。

不过这里也要考虑一点就是：对方系统请求量大不大，如果大到超过自身系统的处理能力，那请求全接下来异步处理可能把系统拖垮。如果时大时小，刚好可以通过actor的邮箱进行平滑处理。所以这也说明实时性要求极高的系统不能应用。

# actor 处理逻辑
DDD也是推崇依赖注入的（当然不用也完全没关系），这里正好使用Spring的Bean定义来初始化一个单例actor。

> 单例actor不是浪费机器能力吗？本来可以多线程处理，为啥非要单线程慢慢跑呢？

如果因为计算能力原因的确需要多actor并行，可以放弃单例依赖注入。但是多actor需要自己控制actor的创建和销毁，actor是不能自行销毁的，必须手动关闭。如果设计不好，频繁创建销毁，性能一般，那还不如使用线程池呢。

> 没有竞态的话使用线程池处理但actor难以应付的场景更好

下面是我定义的bean配置：
```java
@Configuration
public class AkkaSystemConfiguration {

    @Bean
    public ActorSystem getActorSystem() {
        return ActorSystem.create("css-clearing-center");
    }

    @Bean("outSourceLogActor")
    public ActorRef getOutSourceLogActor(ActorSystem system) {
        return system.actorOf(OutSourceLogActor.props(), "outSourceLogActor");
    }

    @Bean("retryClearingActor")
    public ActorRef getRetryClearingActor(ActorSystem system) {
        return system.actorOf(RetryClearingActor.props(), "retryClearingActor");
    }
}
```

这样就可以在上下文中通过Spring获取actor了。

> 如果使用领域编程，在领域对象中没法直接注入bean，因为领域对象自己不是bean。这时候需要通过spring容器获取。比如Hutool-5.1开始提供的工具类cn.hutool.extra.spring.SpringUtil

> 实际上我项目里正是这样获取bean的

但是对于偶尔需要用到的actor，的确可以自行控制你创建和销毁。比如我有一个处理异常数据的actor，由于异常出现的很少，所以在异常数据出现是创建actor，actor的处理逻辑是在finally块中关闭自己：
```java
finally {
    getContext().stop(getSelf());
}
```

# 关于延时消息
有时候处理数据时发现条件还不满足，需要等一会再试。这时候可以发送延时消息给actor。akka提供了两种方法发送延时消息，一种是定时发送一次，一种是以固定频率持续发送。有兴趣的可以百度一下，这里说一下我的一个场景，使用的是第一种方法实现第二种效果：

在收到数据时发现条件尚不满足，于是发送延时1分钟的消息给自身。消息中携带了第一次重试的时间，如果距现在超过一定时间就放弃重试
```java
@Slf4j
public class RetryClearingActor extends AbstractActor {

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .match(RetryClearingBillEvent.class, event -> {
                if (true) { //条件不满足
                    LocalDateTime now = LocalDateTime.now();
                    if (Duration.between(event.getSourceTime(), now).getSeconds() < TimeUnit.MINUTES.toSeconds(30)) {
                        // 超过30分就不再重试
                        getContext().system().scheduler().scheduleOnce(Duration.ofMinutes(1), getSelf(), event, getContext().system().dispatcher(), getSelf());
                    } else {
                        log.warn("超时，放弃自动重试：{}", event);
                    }
                } else {
                    SyncOutSourceToSettlementService.INSTANCE.clearedSync(event.getBizNo(), event.getItemNo(), event.getBillType(), false);
                }
            })
            .build();
    }

}
```

