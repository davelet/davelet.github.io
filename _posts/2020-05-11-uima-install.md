---
layout: post
title: Apache UIMA Sdk 的安装
categories: [dev]
tags: [uima]
---
Apache UIMA 是什么？是一个非结构化文本的分析框架，不过这个框架没有提供太多的个性化工具，而仅仅提供了一套结构和一些基础工具。我了解的不多，所以就不瞎说了。不过既然你看到这篇文章，说明已经了解了一点UIMA（不然为啥过来Google这个主题词呢）。所以闲言少叙，书归正传：这篇说一下UIMA的安装。

# 下载
下载比较简单，页面在 [Downloading Apache UIMA™](http://uima.apache.org/downloads.cgi)。找最新的版本下载，windows下载zip，其他系统下载targz。

下载后解压，然后移动到一个中意的目录，并把目录设置为UIMA_HOME环境变量。

> 下面假设环境变量UIMA_HOME=~/public/apache-uima

# 测试
原则上讲，像上面这样安装完就可以使用了。我们使用官方提供的例子来测试一下。

## 调整文件

进入目录$UIMA_HOME/bin，运行adjustExamplePaths.sh文件。

> 如果是windows环境就运行对应的bat文件。下面也都一样

正常情况下是会报错的，说什么“/bin/sh^M: bad interpreter: No such file or directory”。这是因为文件格式的问题。

使用vim打开adjustExamplePaths.sh，执行:set ff会提示“fileformat=dos”，dos是windows的文件格式，通过:set ff=unix即可修复，然后保存文件。再次运行adjustExamplePaths.sh，会输出一大堆“working on ...”之类的东西，稍等一下执行完。

> 这个过程的目的是调整测试文件，里面会通过java命令运行一个org.apache.uima.internal.util.ReplaceStringInFiles文件。

## 运行分析例子

目录依然切换到$UIMA_HOME/bin下面，执行文件documentAnalyzer.sh。这个文件的目的是调用runUimaClass.sh文件并传参org.apache.uima.tools.docanalyzer.DocumentAnalyzer，后者是一个GUI程序。所以也可以直接运行

```bash
./runUimaClass.sh org.apache.uima.tools.docanalyzer.DocumentAnalyzer
```

> 不过每个sh文件都用编码问题，需要set ff先修复

DocumentAnalyzer对应一个GUI程序：
<div align="center">
<img width="50%" src="/images/post/uima.png">
</div>

有三处需要自己改一下：
 - Input Directory: 改到/examples/data
 - Output Directory: 改到/examples/processed，这个目前需要手动创建（所以可以是其他目录）
 - Location of Analysis Engine XML Descriptor：分析引擎描述符，改到/examples/descriptors/analysis_engine/UIMA_Analysis_Example.xml

点击Run稍等一会就出来一个列表，随便双击打开一个文件可以查看分析结果

<div align="center">
<img width="70%" src="/images/post/uima-res.png">
</div>

比如上图是某个文件只选择显示分析名称（Name）后的结果。