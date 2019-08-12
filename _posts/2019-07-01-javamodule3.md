---
layout: post
title: Java中模块的使用(3) 使用命令行一步步体验
date: 2019-07-01
categories: [dev]
tags: [java, jigsaw]
---

这里我们通过命令行来体验一下如何使用Java的模块。我的机器是Mac，但是其他机器也都一样，只是命令上有稍微的差别。最主要的是要记住使用Java 9+版本。

# 应用结构
通过命令行切换到你期望的工作目录，然后建立一个应用文件夹，比如java-modules:
```
 > mkdir java-modules
 > cd java-modules 
```
简单起见（但是足够了），我们只用到两个模块：一个叫app，一个叫func吧：
```
> mkdir app
> mkdir func
```
func模块会提供一个简单的方法供app模块使用。

这两个模块都是我们的源码，一会我们编译后的文件会放到output目录，为了区别，我们把源码都放到src目录下：
```
> mkdir src  
> mv app src
> mv func src
> mkdir output
```
# 第一个模块
进入到func目录，依次创建包func/math，并创建Java类MyFunc.java:
```
 > cd src/func 
 > mkdir func  
 > cd func    
 > mkdir math     
 > cd math   
 > vim MyFunc.java  
```
MyFunc.java内容如下：
```
package func.math;

public class MyFunc{
    public static void sayHello(){
        System.out.println("Hello Func");
    }
}
```
现在返回模块根目录创建模块描述符:
```
> vim module-info.java
```
内容：
```
module func{
exports func.math;
}
```
这样源码部分写完了，我们来编译这个模块。

返回到应用根目录，执行编译命令：
```
> cd ../.. 
> javac -d output/func src/func/module-info.java src/func/func/math/MyFunc.java
```
> 如果提示说找不到目录，需要先到output下面创建func和app目录

> 如果提示需要class, interface或enum，请核对使用的Java版本

# 第二个模块
进入到src/app目录，依次创建包main/start，并创建Java类MyApp.java:
```
 > cd src/app 
 > mkdir main  
 > cd main    
 > mkdir start     
 > cd start   
 > vim MyApp.java  
```
MyApp.java内容如下：
```
package main.start;

import func.math.MyFunc;

public class MyApp{
    public static void main(String[] a) {
        MyFunc.sayHello();
    }
}
```
这个类引用了上个模块的类，并使用了其中的方法。然后返回src/app目录创建模块描述符:
```
> vim module-info.java
```
内容：
```
module app{
requires func;
}
```
现在来编译这个模块。在应用根目录执行：
```
> javac --module-path output -d output/app src/app/module-info.java src/app/main/start/MyApp.java
```
# 运行
两个模块都编译好了以后，现在运行main方法看一下。
> 如果你在编译过程中遇到问题且不能解决，可以在下面留言。如果看不到留言区，可以点击下面的github图标创建issue

```
> java --module-path output -m app/main.start.MyApp
Hello Func
```
成功输出。

## 命令参数
-d 用来指定目标目录

--module-path 指定模块目录

-m 执行main方法所在类