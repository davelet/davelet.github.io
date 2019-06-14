---
layout: post
title: Java中模块的使用
date: 2019-06-14
categories: [dev]
tags: [java]
---
Java的模块（就是著名的Jigsaw项目的产物）是Java9中开始出现的。
这里介绍一下如何通过IntelliJ IDEA 搭建简单的Java模块项目。我使用的jdk版本是11.0.1。

Jigsaw让Java发生了巨变，甚至要比Java5和Java8带来的变化还大，仅次于Java2的诞生。
以前的版本迭代不过是加点新的API，或者是一些语法糖。
Jigsaw带来的是不仅引入了模块化（和微服务一样），重构了Java源码包（移除了大家总想研究的rt.jar和tools.jar），
而且public关键字的能力降低了，只能模块内可见（颠覆了以前我们的学习内容），
想模块外可见必须使用exports（但它不是关键字，我们命名变量可以使用它）。

按照最初的计划，Jigsaw会在Java7中出现，但是实现太难了，所以延期了（这导致Java7带来的变化太小了，用的人也不多）；
然后承诺Java8中会出现，也食言了，再一次延期（一方面是难，另一方面因为Java8带来的变化也很大）。最终Java9才实现承诺。

尽管Java9带来的变化很大，比如接口私有方法，var关键字等，但是Java9不是一个长期支持版，它和Java10的变更一起在Java11中进行长期支持。
所以骚年们，let's java 11！

目前主流的项目分层一般是三层：主服务层、数据操作层、对外暴露层，一般叫XXX-api，XXX-dao，XXX-server。
我们这里打算搭建一个hello world项目，也分三层，没有数据库层，分别是对外暴露接口层、服务实现层、测试层。
其实两层就够了，一般测试行为是在服务实现层中的。但是两个模块不能完整体现模块的语法，所以特意使用三个模块：

  <div class="entry">
    <img width="30%" src="/images/post/jigsaw.png" alt="diagram" />
  </div>

图中animal是接口模块，pet是实现模块，kennel是测试模块。

# IDEA 搭建
用idea创建一个空的Java项目（什么依赖都不加，什么模板都不用，记得jdk要9+），名称随便，比如j11-jigsaw-101。
  <div class="entry">
    <img width="70%" src="/images/post/ideanew.png" alt="diagram" />
  </div>
创建好以后删掉idea帮我们创建的src目录，然后在项目根上新建模块animal。

> 创建方式有两种：一种是右键项目名称，new -> Module -> 弹出框确定jdk -> next -> 输入模块名称；
> 另一种是单击项目名称，按快捷键command+N，第一个选项就是Module，直接回车往下走。

模块创建好也有src目录，在里面创建一个接口，比如叫“Animal动物”。把这个接口放到一个包里面，比如“com.j11.jigsaw”。

> 要用模块必须把类放在包里

接下来同样创建两个模块，一个是pet一个是kennel。pet里面是实现类，你可以随便实现。
kennel里面是测试类，有main方法。

# 模块
现在你可以运行main方法并且得到期望输出了。

> 咦，不对吧，模块不是应该有个module-info.java文件吗？我们没有创建过啊！为啥能正常编译呢？

在解答这个问题之前，我们先在animal模块上创建module-info文件。点击包内的任务位置，新建module-info.java（和创建package-info.java一样，但是module-info.java会自动放到根目录下）。

接下来应该能看到编译已经报错了。为什么呢？因为模块分为具名模块和不具名模块（实际分了四类：系统模块、应用模块、自动模块、不具名模块），
当没有描述文件的时候，类都加入了不具名模块，能正常引用。创建描述文件后就不能加入不具名模块了，但是又引用不到所以报错了。

> 这段话让我想起了小明说如果不想挂科，宿舍要挂柯南的海报，这叫“挂柯南”；但是如果已经挂了科比的海报就不能挂柯南的了，因为大家都希望“挂科比不挂柯南”。

在animal的模块描述文件中exports相应的包。然后创建其余模块的描述文件，在pet的描述文件中加上requires animal。相应的修复其他依赖文件即可编译成功。

# 传递依赖
在测试中，如果kennel只引入了pet模块就只要requires pet即可。可是如果也使用了animal模块的类呢？

很简单，只要再requires animal就可以了。

可是pet已经引入了animal，kennel引入了pet却还要再引入animal一次不是很繁琐吗？

Java还提供了一个关键字transitive（和exports一样只是关键字不是保留字），在pet引入animal的时候使用requires transitive animal就可以保证下一级能够继续使用（这也是我创建三个模块的原因）。

# source code
完整的项目代码地址：[j11-jigsaw-101](https://github.com/davelet/j11-jigsaw-101)