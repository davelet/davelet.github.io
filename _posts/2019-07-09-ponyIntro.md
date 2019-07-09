---
layout: post
title: Pony语言简介
date: 2019-07-09
categories: [dev]
tags: [pony, julia]
---
Pony语言，或者称为ponylang，是一门小众语言。诞生了有两三年，但是一直不温不火。它是基于Actor模型创建的语言，所以能够使用Actor模型的场景可以使用它。比如事件驱动、高并发、低延迟等。不过它的前辈，smalltalk、erlang、akka模型，没有一个🔥的，它想🔥还十分艰难。

# 特性

## 类型安全
真的很安全，不信的话...我拿一篇数学论文证明给你看：[Deny Capabilities for Safe, Fast Actors](https://www.ponylang.io/media/papers/fast-cheap-with-proof.pdf)。

## 内存安全
不好有野指针或者内存溢出。Pony也没有null的概念！

## 异常安全
这个“异常”是抛异常的那个异常，不是“非常”的意思😸。Pony没有运行时异常，所有的异常都定义了语义，总是会被捕获。

## 没有竞态
Pony没有锁、原子操作，或与之类似的东西。相反，它的类型系统保证了并发程序也不会有竞态。所以可以随便写高并发程序都不会有问题。

## 没有死锁
连锁都没有，当然不会死锁咯！死锁这东西在Pony中绝对不会出现。

## 本地执行
Pony使用了提前编译（AOT)，没错就是安卓也在用的那个。所以不用解释执行也不依赖虚拟机。（不过编译是真慢，llvm的锅？） 

## 兼容C/C++
Pony和C可以互相调用（啥，C还能调用Pony？C认识Pony是谁？没错，C没法直接调用，Pony会自己生成头文件供C调用）。

# 结论
看起来不错吧，不过我还是不用，我用Julia😁