---
layout: post
title: springboot项目中简单使用akka
date: 2019-07-03
categories: [dev]
tags: [java, akka, springboot]
---
Spring体系是Java开发中用的最多的框架，目前主要都用springboot。Akka是基于事件驱动的Actor模型，使用Scala开发，但也提供了Java版本的库。
这里简单演示一下如何在Spring项目中使用Akka。

# 搭建一个spring mvc项目
为了使用方便，可以搭建一个spring mvc的项目。这样可以通过swagger或者http客户端调试。

> 也可以搭建一个dubbo项目，然后通过telnet调试。不过还需要注册中心，比较麻烦。如果有现成的直接用倒是可以

这个不说怎么搭建了。如果没有现成的项目，直接使用springboot的样例项目就行。

# 使用Akka

Akka提供给Java的pom坐标有点怪，artifactId是以一个版本号结尾，但和maven的版本号又不同。
这里使用的是以2.12结尾的artifactId，建议你也使用一样的。version没关系，使用哪个都行：
```
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor_2.12</artifactId>
    <version>2.5.22</version>
</dependency>
```
Akka需要使用ActorSystem，我们给它提供一个bean：
```
@Configuration
public class AkkaConfiguration {
    @Lazy
    @Bean
    public ActorSystem actorSystem() {
        return ActorSystem.create("your-favarite-name");
    }
}
```
接下来创建两个Actor，其中一个使用注解@Component标注，可在Controller中引用：
```
@Slf4j
public class Teller extends AbstractActor {
    static Props props() {
        return Props.create(Teller.class, Teller::new);
    }

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .matchAny(
                        f -> log.info("obj is {}", f))
                .build();
    }
}

@Component
public class Tellee {
    @Autowired
    private ActorSystem system;

    public void tell() {
        final ActorRef teller = system.actorOf(Teller.props(), "teller");
        teller.tell("told you that", ActorRef.noSender());
    }
}
```
在控制器中引用Tellee的bean，并调用其tell()方法。启动项目并模拟调用，可看到输出“obj is told you that”。

---

参考资料：[https://blog.csdn.net/p_programmer/article/details/85041603](https://blog.csdn.net/p_programmer/article/details/85041603)
