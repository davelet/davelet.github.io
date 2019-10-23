---
layout: post
title: akka Streams入门（Java版）
categories: [dev]
tags: [akka, actor]
---

Akka 组件不少，最基础的当然是akka-actor模块。akka-streams模块是用作流处理的，“流”是什么？流是数据集合的一种处理方式，我们用了这么久的Java8 Stream api，应该知道它是用来处理Iterator数据结构的。akka-streams 除了以流式处理外，还提供了响应式设计、背压逻辑等。这里我们看一个小例子，只是熟悉它的api，并不涉及原理。

这个例子就是把一组整型数两两分隔形成多组，每组求一下均值。比如输入是1,2,3,4,5，输出就是1.5,3.5,5。因为最后只剩5了，均值就是它自己。

---
## 准备
使用前需要引入依赖：
```
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-stream_2.11</artifactId>
    <version>2.5.2</version>
</dependency>
```
因为是在akka体系，所以需要注册一个ActorSystem：
```
private static ActorSystem actorSystem = ActorSystem.create("testActor");

```
## 第一步
我们把输入的字符串解析成整数列表：注意这里使用的是Java8的 Stream
```
private List<Integer> parseLine(String line) {
    String[] fields = line.split(";");
    return Arrays.stream(fields)
            .map(Integer::parseInt)
            .collect(Collectors.toList());
}
```
下面开始使用akka streams处理。
## akka streams api

- Source：源是生成或者说是提供数据流的，是akka streams 处理的入口
- Flow：流是处理过程，有一个输入和一个输出，是延迟执行的
- Sink：下沉，是Flow的中止操作。和Java类似，akka的流操作也分中间操作和中止操作
- Materializer：真正执行逻辑的工厂，它会生成一个actor去处理下沉。不用的话就传一个NotUsed

## 第二步
通过第一步的方法生成流：
```
private Flow<String, Integer, NotUsed> parseContent() {
    return Flow.of(String.class)
            .mapConcat(this::parseLine);
}
```
生成的Flow有三个泛型参数，第一个是输入类型，第二个是输出的集合元素类型，第三个是Materializer。
这里我们输入一个字符串，生成的是整型集合。

## 第三步
把第二步的整型集合两两分组分别求均值：
```
private Flow<Integer, Double, NotUsed> computeAverage() {
    return Flow.of(Integer.class)
            .grouped(2) // 从前往后每两个一组
            .mapAsyncUnordered(8, integers ->
                    CompletableFuture.supplyAsync(() -> integers.stream()
                            .mapToDouble(v -> v)
                            .average()
                            .orElse(-1.0)));
}
```
可以看出它的输出是Double集合，因为每个均值都转为Double了。
另外这里使用了mapAsyncUnordered方法，这是一个并行执行的无序方法，第一个整数参数代表通过几个线程执行，后面是我们通过Java API求均值的实现。

## 第四步
把上面两个过程合成一个流。可以看到第三步并没有调用第二步的方法，那怎么拿到其输入呢？通过Flow的via方法：
```
private Flow<String, Double, NotUsed> calculateAverage() {
    return Flow.of(String.class)
            .via(parseContent())
            .via(computeAverage());
}
```
注意到这个Flow的输入是String，输出是Double集合。

## 第五步1
加入说处理以后要把处理结果写入数据库，我们通过一个CompletableFuture异步写入：
```
private CompletionStage<Double> save(Double average) {
    return CompletableFuture.supplyAsync(() -> {
        // 假装写入数据库
        System.out.println(average);
        return average;
    });
}
```

### 第五步2
然后通过流包装上面的异步过程，并生成下沉：
```
private Sink<Double, CompletionStage<Done>> storeAverages() {
    return Flow.of(Double.class)
            .mapAsyncUnordered(4, this::save)
            .toMat(Sink.ignore(), Keep.right());
}
```
先生成了一个流，输入是Double；并再次使用了mapAsyncUnordered方法通过4个线程执行保存过程。
然后将流指向下沉，使用toMat方法。

## 第六步
执行完整的过程。首先将输入生成源，通过via方法把源转为流，通过runWith方法执行下沉操作：
```
private CompletionStage<Done> calculateAverageForContent(String content) {
    return Source.single(content)
            .via(calculateAverage())
            .runWith(storeAverages(), ActorMaterializer.create(actorSystem))
            .whenComplete((d, e) -> {
                if (d != null) {
                    System.out.println("Import finished ");
                } else {
                    e.printStackTrace();
                }
            });
}

public static void main(String[] args) {
    CompletionStage<Done> stage = calculateAverageForContent("1;9;11;0;6;3;100");
    stage.whenComplete((a, b) -> {
        System.out.println(a.getClass());
    });
    stage.thenRun(() -> actorSystem.terminate());
}
```

# 提示
Java开发者查看akka的源码有点懵，这样连注释也没法看。Mac上有一个免费的文档软件叫Dash，windows也有类似的软件。可以把akka的文档下载下来查询，方便多了。

# 代码

[https://gist.github.com/davelet/5820db5d6a4048fd1d6f5b20d30d4e22](https://gist.github.com/davelet/5820db5d6a4048fd1d6f5b20d30d4e22)