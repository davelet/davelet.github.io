---
layout: post
title: Transfer Learning (迁移学习) in TensorFlow
---

这篇文章我打算分享一下自己学习用TensorFlow进行图片分类的经历。

Here I shall explain the process of my learning on how to classify images by TensorFlow.

刚开始学习用tensorFlow进行图像识别时，先接触到的概念是“物体识别”，或叫“对象识别”。通过官方的例子，看到tf可以给图片中的人和风筝打标记。阅读了很多材料，主要是官方的介绍和网友们的一些尝试，比如《[TensorFlow object detection API应用](https://www.cnblogs.com/zongfa/p/9663649.html)》，我对物体识别的认识还停留在“采集样本-标记数据-训练”的路子上。

At first I ran into "Object Detection" by tensorFLow. What I mastered then is "object detection = image classification".

在网上找到一个哥们在Windows下的经历，写得很详细。于是我把它翻译过来，并移植到了Mac上。如果你也用的Mac，可以来“[如何在Mac上使用TensorFlow对象识别API进行多对象识别训练？](https://github.com/davelet/TensorFlow-Object-Detection-API-Tutorial-Train-Multiple-Objects-On-Macos)”看看；如果你用的是windows系统，上面的链接一开始也给出了原文的地址。总之，你应该来我的github库看看:D

I translated a github repo into Chinese. The original repo tells how to train a multiple objects detector using tensorFLow under Windows OS while I put it onto Macos. Please feel free to access my repo at [TensorFlow-Object-Detection-API-Tutorial-Train-Multiple-Objects-On-Macos](https://github.com/davelet/TensorFlow-Object-Detection-API-Tutorial-Train-Multiple-Objects-On-Macos).

这个库里面分成了8步。第二步结束的时候应该能够把狗狗、风筝、人都识别出来。这说明你的环境搭建成功了。就好像你要用Java开发复杂的系统一样，走到这里说明你的jdk安装好了，并且通过`javac`、`java`命令能够打印"hello world"出来。

There are eight sections in the repo. If you managed to finish the 2nd section, you can yell out "Yoohoo"! The result of the 2nd section shows that your tensorFlow object detection environment is done.

后面几步是训练一个模型能够识别几张扑克牌：九、十、钩、圈、凯、尖。这几张牌是“Pinochle Deck”游戏中所用到的扑克牌。我的2018 Mac Pro跑了几十个小时，训练了近3万步，识别那几张扑克还是有错误的。你如果在训练，不用紧张：训练过程中每10分钟会记录一下进度。所以你随时可以中断你的训练，随时可以用同样的命令回复训练。tf会自动继续之前记录的进度。当然了，最好是在刚好记录进度后暂停，不然有一点训练内容没记录就丢失了。

The rest of the repo teaches how to train a model to recognize poke of from nine to ace. It may take you a very long time (like days); However, don't be nervous, you can paused the process anytime by shutting down the training, and you can restore the training anytime by re-run the training command.
> 未完待续 To Be Continued...


