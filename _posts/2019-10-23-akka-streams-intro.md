---
layout: post
title: akka Streams入门（Java版）
categories: [dev]
tags: [akka, actor]
---

Akka 组件不少，最基础的当然是akka-actor模块。akka-streams模块是用作流处理的，“流”是什么？流是数据集合的一种处理方式，我们用了这么久的Java8 Stream api，应该知道它是用来处理Iterator数据结构的。akka-streams 除了以流式处理外，还提供了响应式设计、背压逻辑等。这里我们看一个小例子，只是熟悉它的api，并不涉及原理。

这个例子就是把一组整型数两两分隔形成多组，每组求一下均值。比如输入是1,2,3,4,5，输出就是1.5,3.5,5。因为最后只剩5了，均值就是它自己。

---

因为是在akka体系，所以需要注册一个ActorSystem：
```
private static ActorSystem actorSystem = ActorSystem.create("testActor");



    public static void main(String[] args) {
        AkkaConfiguration configuration = new AkkaConfiguration();
        CompletionStage<Done> stage = configuration.calculateAverageForContent("1;9;11;0;6;3;100");
        stage.whenComplete((a, b) -> {
            System.out.println(a.getClass());
        });
        stage.thenRun(() -> actorSystem.terminate());
    }

    private List<Integer> parseLine(String line) {
        String[] fields = line.split(";");
        return Arrays.stream(fields)
                .map(Integer::parseInt)
                .collect(Collectors.toList());
    }

    private Flow<String, Integer, NotUsed> parseContent() {
        return Flow.of(String.class)
                .mapConcat(this::parseLine);
    }

    private Flow<Integer, Double, NotUsed> computeAverage() {
        return Flow.of(Integer.class)
                .grouped(3)
                .mapAsyncUnordered(8, integers ->
                        CompletableFuture.supplyAsync(() -> integers.stream()
                                .mapToDouble(v -> v)
                                .average()
                                .orElse(-1.0)));
    }

    private Flow<String, Double, NotUsed> calculateAverage() {
        return Flow.of(String.class)
                .via(parseContent())
                .via(computeAverage());
    }

    private CompletionStage<Double> save(Double average) {
        return CompletableFuture.supplyAsync(() -> {
            // write to database
            System.out.println(average);
//            throw new RuntimeException(average + "");
            return average;
        });
    }

    private Sink<Double, CompletionStage<Done>> storeAverages() {
        return Flow.of(Double.class)
                .mapAsyncUnordered(4, this::save)
                .toMat(Sink.ignore(), Keep.right());
    }

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

```