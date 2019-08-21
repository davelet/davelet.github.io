---
layout: post
title: 使用 TensorFlow 进行图片识别的例子（Mac平台下）
categories: [dev]
tags: [tensorflow]
---

# python3 环境的安装
直接brew install python3就可以了。如果没有brew先要安装brew，随便查一下怎么安装都能找到。

> 这里可能会有点问题：我之前用安装包安装过又卸载了，但是没卸载干净，导致brew链接失败。
不过brew会提示你删掉什么目录就可以了，我大概删了十几个目录，又执行brew link python就可以了。

安装好以后可以通过python3 -V验证。

# 搭建虚拟环境

python开发推荐在虚拟环境venv中进行。

先创建工作目录，比如在~/virtualenvs下面，就先
> mkdir ~/virtualenvs
 
创建一个环境，比如myvenv：
> python3 -m venv ~/.virtualenvs/myvenv

这个环境是给python3用的，所以里面的python命令不是python2的。进入这个环境：
> source ~/.virtualenvs/myvenv/bin/activate
退出环境：
> deactivate

虚拟环境执行的操作，比如安装其他软件，只对当前虚拟环境可用。