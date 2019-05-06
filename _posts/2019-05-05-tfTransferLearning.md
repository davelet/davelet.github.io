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

在完成了图片的识别之后，你可能有点迷茫：一方面你照猫画虎学会了训练对象识别模型，另一方面烦长的采集和标记工作让你望而却步。甚至你电脑的配置也低得让你认识到该换一台好一点的windows机器了。为了提升你的兴趣，可以尝试一下摄像头的识别。下载[webcam.py](https://github.com/davelet/yh-ml-learning-group-201903/blob/master/%E5%AD%A6%E4%B9%A0%E8%BF%9B%E5%BA%A6/%E9%AD%8F%E6%99%93%E4%B8%9C/webcam.py)到tf对象识别的跟目录。第一次跑的话把36-42行的注释取消，需要先下载训练好的模型。代码运行起来以后会通过电脑的摄像头识别物体，按q退出。

Here is a python script file in my another github repo: [webcam.py](https://github.com/davelet/yh-ml-learning-group-201903/blob/master/%E5%AD%A6%E4%B9%A0%E8%BF%9B%E5%BA%A6/%E9%AD%8F%E6%99%93%E4%B8%9C/webcam.py). It can detect objects by computer's camera. 

之后我尝试了水果的识别训练。具体的方法在我的另一个github库上：[苹果、橘子、香蕉识别](https://github.com/davelet/ABO-detector)。这个库提供的信息有限，因为如果你能训练扑克的识别、花的识别、红绿灯的识别，没有理由不能训练水果的识别。反正我训练好以后，识别正确率比上面的扑克高多了。这个库的最后提供了对模型转换极其重要的信息：inception模型和mobilenet模型的输入输出张量叫什么名字。为了获取模型的张量名称，我查阅了大量资料才偶然的找到。

For model converting, the inception model's input and output array names are "Placeholder" and "final_result", while the mobilenet's are "input_image" and "MobilenetV1/Predictions/Softmax". For more infomation plase refer to my repo of [ABO-detector](https://github.com/davelet/ABO-detector).

---
有了上面训练的模型以后，我开始考虑如何把模型放到移动设备上，比如我的安卓手机。经过漫长的资料查询，还是没有找到能成功把pb文件转换成tflite文件的方法。百无聊赖地，我开始阅读tf lite的官方文档，了解到了“图片分类”。

So I tried to convert the .pb file to .tflite to install on my Android phone, and finally failed. I started reading official documents of tf lite comprehensively, and finally succeded: I found the way of "image classification".

图片分类用到的技术叫“迁移学习”，即用已经训练好的模型去训练图片集，只需要用很短的时间就能训练完成，并且识别率很好。我用迁移学习技术训练上面提到的水果图片，几分钟就训练完了，而且识别率极高。

The core technology is "transfer learning" in images classification. It can reduce the training time extraordinarily.

图片分类和对象识别不同的是，对象识别需要标记；而图片分类会把整张图片当成某一个东西。所以图片分类只要收集图片就行，不用再标记了。这大大提升了作业流程。详细的教程请看我的另一个github库“[Mac下安卓应用的tensorflow迁移学习](https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos)”。我现在已经可以很方便的进行迁移学习。

The whole training and converting details are shown in my github repo of [tensorflow_transfer_learning_for_android_on_macos](https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos).

——魏晓东 完成于2019-05-06 23:20:03