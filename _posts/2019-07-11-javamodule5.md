---
layout: post
title: Java中模块的使用(5)服务加载器
date: 2019-07-11
categories: [dev]
tags: [java, jigsaw]
---
Java的ServiceLoader（java.util.ServiceLoader，服务加载器）不是Java9中才出现的，Java6就开始使用了。这里的服务，可以简单的理解为Java接口，加载器加载的是接口的实现类（也可以说成服务是实现了接口的类）。在Java9以前主要用于实现spi机制，也就是动态加载实现类。Java9赋予了它新的功能，这篇文章简单讲一下ServiceLoader的使用。

> 面向对象编程会将how和what分开，所以有接口和实现：接口定义了what，实现阐明了how。

# SPI 机制
关于spi的介绍网上能搜出来一大把，很多还讲得不错。

如果你懒得去搜，这里简单演示一下它的用法。

## 定义服务接口
```
package com.j11.spi;
public interface IMovable {
    void move();
}
```
这里定义了一个接口 IMovable，表示一种可以移动的特征。

## 编写多个实现类
实现类可以有一个或多个。为了演示，这里实现两个：
```
package com.j11.spi;
public class Human implements IMovable {
    @Override
    public void move() {
        System.out.println("human steps...");
    }
}

package com.j11.spi;
public class Car implements IMovable {
    @Override
    public void move() {
        System.out.println("Car wheels...");
    }
}

```
## 服务发现
ServiceLoader要求被加载服务的描述符文件必须在/META-INF/services目录下。创建该目录，并在该目录下创建文件com.j11.spi.IMovable，内容如下：
```
com.j11.spi.Human
com.j11.spi.Car
```
每个实现类写一行。

## spi
SPI全称Service Provider Interface，即服务提供方接口，创建一个main方法类，方法体如下：
```
ServiceLoader<IMovable> shouts = ServiceLoader.load(IMovable.class);
for (IMovable s : shouts) {
    s.move();
    if (s instanceof Car) {
        System.out.println("match");
    }
}
```
输出
```
human steps...
Car wheels...
match
```
>IDEA 内置对spi机制的支持，如果接口和描述符不匹配则会立即警告

# Java模块对spi的支持
Java9+中，依然可以写出上面这种代码。不过显示模块化的话，可以使用provides、uses和with关键字（注意，它们和其他的模块关键字一样都不是保留字，可以用做变量名称），provides就是spi中的p。

## 创建服务接口模块
创建一个模块，只包含服务接口和服务提供接口：
```
package com.j11.spi;

public interface IService {
    String getName();

    void serve();

    void halt();
}

package com.j11.spi;

public interface IServiceProvider {
    IService provide();
}

module service {
    exports com.j11.spi;
}
```

## 创建接口实现模块
创建模块实现接口：
```
package com.j11.service.impl;

import com.j11.spi.IService;
public class Server1 implements IService {
    @Override
    public String getName() {
        return "s1";
    }

    @Override
    public void serve() {
        System.out.println("s1 is serving");
    }

    @Override
    public void halt() {
        System.out.println("s1 stopped");
    }
}

package com.j11.service.impl;

import com.j11.spi.IService;
public class Server2 implements IService {
    @Override
    public String getName() {
        return "s2";
    }

    @Override
    public void serve() {
        System.out.println("s2 is serving");
    }

    @Override
    public void halt() {
        System.out.println("s2 stopped");
    }
}
```
服务提供方实现：
```
package com.j11.provider.impl;

import com.j11.service.impl.Server1;
import com.j11.spi.IService;
import com.j11.spi.IServiceProvider;
public class S1Provider implements IServiceProvider {

    @Override
    public IService provide() {
        return new Server1();
    }
}

package com.j11.provider.impl;

import com.j11.service.impl.Server2;
import com.j11.spi.IService;
import com.j11.spi.IServiceProvider;
public class S2Provider implements IServiceProvider {
    @Override
    public IService provide() {
        return new Server2();
    }
}
```
模块描述符中暴露服务，使用provides IX with Y,Z：
```
module provider {
     requires transitive service;

     provides com.j11.spi.IServiceProvider with com.j11.provider.impl.S1Provider, com.j11.provider.impl.S2Provider;
}
```

## 创建应用模块
应用模块要使用服务，用到了uses关键字：
```
module customer {
     requires provider;
     uses com.j11.spi.IServiceProvider;
}
```
启动应用：
```
package com.j11.spi.customer;

import com.j11.spi.IService;
import com.j11.spi.IServiceProvider;

import java.util.ServiceLoader;
public class Customer {
    public static void main(String[] args) {
        ServiceLoader<IServiceProvider> sl = ServiceLoader.load(IServiceProvider.class);

        for (IServiceProvider provider : sl) {
            IService server = provider.provide();
            server.serve();
            if (server.getName().equals("s2")) {
                server.halt();
            }
        }
    }
}
```
打印结果：
```
s1 is serving
s2 is serving
s2 stopped
```

# 源码

[https://github.com/davelet/jigsaw-spi-101](https://github.com/davelet/jigsaw-spi-101)