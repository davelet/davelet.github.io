---
layout: post
title: Lombok插件不支持IDEA 2018的解决方案
categories: [dev]
tags: [ide]
---

如果你也在intellij idea 2018中使用了Lombok，现在的插件是不支持的。虽然打包和build都不抱错，但是打开编辑器看到一大片红色的编译错误提示还是十分不爽，关键是没有了自动完成。这里说一下解决方案。

> 下面是MAC的解决方法，其他系统可能有不同

 - git clone https://github.com/mplushnikov/lombok-intellij-plugin.git
 - 在gradle.properties 文件中设置ideaVersion=2018.1
 - 运行命令 ./gradlew buildPlugin
 - 打开你的idea
 - 卸载掉 Lombok plugin
 - 通过本地安装插件build/distributions/lombok-plugin-0.17.zip
 - 重启idea


这个过程虽然不需要vpn，不过gradle会默认访问maven的中心仓库，所以有些慢。两三个小时搞定就算很快了。而且它要下载idea的源码，所以慢慢等吧。

