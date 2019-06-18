---
layout: post
title: Java中模块的使用(2) 
date: 2019-06-17
categories: [dev]
tags: [java]
---

在Java9以前，maven已经提出了模块的概念。用过maven的对于maven的模块绝对不陌生。那么现在有了jigsaw，还需要maven的模块吗？本篇简单介绍一下Java模块和maven的集成使用。通过本篇可以看到，Java模块和maven模块是两个维度的东西，互不干涉对方内政。

# 用Java模块搭配maven模块
> 希望你对如何创建maven项目很熟悉，因为下面有些细节我不会详细讲。

> 我使用的是Java11

用IDEA创建一个空的maven项目，比如
```
<groupId>com.j11</groupId>
<artifactId>jigsaw-maven</artifactId>
```

在根项目右键，给项目增加一个maven模块（也是idea模块），比如
```
    <parent>
        <artifactId>jigsaw-maven</artifactId>
        <groupId>com.j11</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>helloModule</artifactId>
```
模块信息会加到父级的pom.xml中。

在helloModule模块的src/main/java目录下随便创建一个类，比如com.j11.hello.HelloModules。我们的目的只是要在另一个模块引用它。

在当前模块创建Java模块描述文件，并exports com.j11.hello包。

接下来用同样的方式创建第二个maven模块，比如
```
    <parent>
        <artifactId>jigsaw-maven</artifactId>
        <groupId>com.j11</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>okModule</artifactId>
```
创建module-info.java并requires com.j11.hello。

在okModule的src/main/java目录下随便创建一个类，比如
```
package com.j11.ok;

import com.j11.hello.HelloModules;

public class OkYou {
    public static void main(String[] args) {
        System.out.println("ok module");
        var var = new HelloModules();
        System.out.println(var.getClass());
    }
}
```

运行，可看到输出结果。
# 结论
你可以尝试把okModule的模块描述文件删掉，看下运行是否正常。

从上面的过程可以看到：和maven搭配的时候，Java的模块是用来控制访问级别的，前面说过public关键字的能力降低到模块内了。如果用到了Java模块（没用到就和Java9之前一样），就必须通过requires才能访问，及时它们在同一个maven项目内。maven是用来组织项目的，它的能力和使用Java模块以前一样：管理依赖，构建项目。不适用maven，在任何Java版本下这都是一件费力气的事情。

# 源码

[jigsaw-maven](https://github.com/davelet/jigsaw-maven.git)