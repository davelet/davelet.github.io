---
layout: post
title: Java中模块的使用
date: 2019-06-14
categories: [dev]
tags: [java]
---
Java的模块（就是著名的Jigsaw项目的产物）是Java9中开始出现的。
这里介绍一下如何通过IntelliJ IDEA 搭建简单的Java模块项目。我使用的jdk版本是11.0.1。

目前主流的项目分层一般是三层：主服务层、数据操作层、对外暴露层，一般叫XXX-api，XXX-dao，XXX-server。
我们这里打算搭建一个hello world项目，也分三层，没有数据库层，分别是对外暴露接口层、服务实现层、测试层。
其实两层就够了，一般测试行为是在服务实现层中的。但是两个模块不能完整体现模块的语法，所以特意使用三个模块：

  <div class="entry">
    <img width="60%" src="https://github.com/davelet/pic_ref_lib/blob/master/201906/jigsaw-diagram.png" alt="diagram" />
  </div>

图中animal是接口模块，pet是实现模块，kennel是测试模块。

# IDEA 搭建
coming soon...