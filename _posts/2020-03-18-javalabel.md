---
layout: post
title: Java中标签的用法（你以为是用在嵌套循环中的吗？）
categories: [dev]
tags: [java]
---

Java 中的标签怎么用？估计大部分人都会说出来：“用于在多层循环中，从内层循环跳出外层循环！”的确，我最早接触标签的用法也的确是这样用的。

> 遥想当年自学Java，遇见问题没有算法概念，经常三四层循环嵌套，在javaeye上问到了如何从内层跳出外层

> 提到javaeye，又免不了伤感，尤其最近又在上面看到范凯、R大等人当年活跃的痕迹。[一起缅怀吧](/helloGitPage/)!

# 一、循环上的标签

如果你现在还不了解Java的标签，我先简单介绍一下循环中的标签如何用。

比如有一个三层循环，我们想要在最内层满足某个条件的时候结束整个循环，该怎么写？

## 不使用标签
正常情况下不使用标签是实现不了的，但是如果循环体后面没有其他代码了，那么我们可以使用 return :

```java 
void notLabel(int max) {
    var res = 0;
    for(var a = 1; a < 10; a++) {
        for(var b = a; b < 10; b++) {
            for(var c = b; c < 10; c++) {
                res += a * b * c;
                if (res > max) {
                    return res; // 这里返回不影响逻辑，因为后续没有代码需要再执行
                }
            }
        }
    }
}
```
## 使用标签
如果不能提前返回，就需要使用标签控制了：
```java 
void notLabel(int max) {
    var res = 0;
    第一个标签: // 标签和变量名一样，最好不用中文
    for(var a = 1; a < 10; a++) {
        第二个标签:// 这个标签和下面的标签用不上
        for(var b = a; b < 10; b++) {
            第三个标签:
            for(var c = b; c < 10; c++) {
                res += a * b * c;
                if (res > max) {
                    break 第一个标签; // 跳出整个循环
                }
            }
        }
    }
    // 其他处理
    handle(res);
}
```
通过break或continue后面跟上标签名，可以自由控制要跳出和继续那一层循环体。

## 扩展
### 可以使用return的场景
上面介绍了不用label而用return的场景。但是我们依然可以使用lable而不用return。
哪种性能更好呢？

实际上，对于可以使用return而没有使用的地方，编译器在进行优化时会判断出来应该使用return，进而替你使用return。
### 两层循环中的标签
```java 
void notLabel(int i) {
    label:
    for(var b = 1; b < 10; b++) {
        var res = 0;
        for(var c = b; c < 10; c++) {
            res += b * c;
            if (res == i) {
                break label;
            }
            if (res > 100) {
                continue label; // 使用break代替contine label
            }
        }
    }
}
```
上面的代码没什么问题。但是在两层循环中想要继续执行外层循环，只要在内层break即可，无需contine label。

和上面一样，使用哪一个都一样，因为编译器在优化时依然使用break代替continue。

# 二、代码块上的标签

一次偶然的机会不小心写出了如下“本该编译报错”的代码结构运行良好：
```text
boolean notLabel() {
    url://www.manxi.info
    return true;
}
```

百度了好久依然没找到为何可以编译。打开谷歌命中的第一条就解答了我的疑惑：[java label usage](https://stackoverflow.com/questions/19836549/java-label-usage)。

所以除了循环，标签还可以用在语句和代码块上。不过用在语句上没什么用，我们也没法goto。用在代码块上可以在块内中止块的执行：
```java
void notLabel(int a) {
    label:
    {
        System.out.println(1);
        if (a == 0) {
            break label;
        }
        System.out.println(2); // 如果输入是0，这一句就执行不到了
    }
}
```

这样，我们就有了另一种控制流程的方式，你可以试一下。