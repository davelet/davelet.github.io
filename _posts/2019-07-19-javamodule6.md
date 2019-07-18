---
layout: post
title: Java中模块的使用(6)高级模块用法
date: 2019-07-19
categories: [dev]
tags: [java, jigsaw]
---
Java模块系统是给Java语言上增加的一项十分完善的特性。可能正是由于它是如此的丰富和完善（如之前介绍的用于兼容老代码的非具名模块，更加细腻的访问控制等），才一直从原定的Java7中拖延到了Java9中。本文介绍一点模块中的高级场景。

正如maven项目中的依赖关系，模块中也有依赖层次，就是模块图。
<div align="center">
<img width="50%" src="/images/post/jmg.png">
</div>

上图中A模块依赖BC模块，B模块依赖DE模块，C模块依赖F模块。当要编译A模块的时候，编译器就要确定依赖层次，进而构造出上面这张模块图。在某些场景下，我们需要用到三种特殊的模块操作来简单有效的指明模块关系。
# open module
前面我们知道了要暴露包内文件需要使用exports关键字来导出模块，但有时候编译期并不需要依赖某个模块，只有运行期才用得上（比如反射）。

假设我们有一个模块，里面有两个包，里面分别有一个类：
```
package com.j11.exports
public class Exports {
    public void export() {
        System.out.println("export");
        Open open = new Open();
        System.out.println(open);
    }
}

package com.j11.open;
public class Open {
    @Override
    public String toString() {
        return "hi there";
    }
}
```
当描述符如下时：
```
module openModule {
     exports com.j11.exports;
}
```
很明显只有com.j11.exports.Exports 可以被requires到。但是如果在运行期要用到却访问不到是不行的，比如上面看到 Exports 引用了 Open 类，如果调用其export()方法就会报错：
```
java.lang.IllegalAccessException: class com.j11.main.Main (in module mainModule) cannot access class com.j11.open.Open (in module openModule) because module openModule does not export com.j11.open to module mainModule
```
为了克服这种意外，可以在描述符中使用 open 关键字：
```
open module openModule {
     exports com.j11.exports;
}
```
使用open后模块内所有的public类型都可以在运行时访问，但只有明确exports的包才能在编译器访问。

# opens 语句
除了使用open module，还有一种更加精细的运行时访问控制，就是使用opens关键字：
```
module openModule {
     exports com.j11.exports;
     opens com.j11.open;
}
```
这样com.j11.exports包可以在编译器访问，com.j11.open可以在运行期访问；如果有其他包则任何时候无法访问。

> 注意：不能在open module中再用opens，否则编译不了

opens 语句中，可以在包名后面使用to来指定只有哪个模块可以在运行时访问该包。这和 exports 一样。

# requires static 语句
requires 进来的模块可以同时在编译期和运行期访问。如果只想在编译器使用，可以在requires 后面增加 static 关键字。比如模块a依赖了模块b但是并不使用b其中的类，另一个模块c依赖了a模块也使用了b模块的类。这样模块a的图中并不包含模块b。

> 只在编译期使用的模块一般有：在IDEA中使用JetBrains的注解，FindBugs的注解，Lombok等。