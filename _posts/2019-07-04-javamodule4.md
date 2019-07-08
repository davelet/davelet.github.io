---
layout: post
title: Java中模块的使用(4) 
date: 2019-07-04
categories: [dev]
tags: [java, jigsaw]
---

在Java9以前，我们常有这种“痛楚”：写出来的类只想给应用内其他包中的类使用，但是用于声明为public的，导致任何一个类都可以引用。
java的模块系统可以帮我们避免这种情况。这里简单介绍一下如何更加严格的控制访问。

# 创建项目
和往常一样，你可以用任何工具，包括notepad或终端里的vim，创建一个Java项目。
我已经创建好了三个模块，你可以下载源码：[j11-jigsaw-exports-to](https://github.com/davelet/j11-jigsaw-exports-to)。
<div align="center">
    <img width="40%" src="/images/post/0704.png">
</div>
上面是这个项目的结构，可以看到三个模块分布得就像他们的名字一样简单清晰。

第一个模块的描述符是这样的：
```
module firstModule {
     exports com.j11.first;// to secondModule;
}
```
后面两个模块都依赖了这个模块，并且都有自己的main方法，可以直接运行。

# 访问控制
接下来编辑第一个模块的描述文件，将其修改成
```
module firstModule {
     exports com.j11.first to secondModule;
}
```
再次编译运行两个main方法，可以看到第三个模块不能编译：
<div align="center">
    <img width="90%" src="/images/post/0704-1.png">
</div>
而第二个模块运行正常。

# 结论
在导出模块exports的时候，可以使用to关键字指明导出的模块只能供哪个模块使用。当然可以指定多个模块：
```
module firstModule {
     exports com.j11.first to secondModule, thirdModule;
}
```