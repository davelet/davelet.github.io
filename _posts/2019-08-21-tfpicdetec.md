---
layout: post
title: 使用 TensorFlow 进行图片识别的例子（Mac平台下）
categories: [dev]
tags: [tensorflow]
---
这里演示一下如何通过安卓手机识别训练好的花朵。这些花朵都是官方例子里的。我的电脑是Mac，所以下面的方法可能不适用与windows平台。

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

# TensorFlow环境搭建
先通过pip安装tensorflow，在虚拟环境里面执行
> pip install tensorflow

然后安装protobuf，protobuf是TensorFlow使用的字节流框架。下载[https://github.com/protocolbuffers/protobuf/releases/download/v3.9.1/protobuf-all-3.9.1.tar.gz](https://github.com/protocolbuffers/protobuf/releases/download/v3.9.1/protobuf-all-3.9.1.tar.gz)到任意目录并解压。
进入解压目录执行
> ./autogen.sh && ./configure && make

如果失败了尝试执行
> brew install autoconf && brew install automake
后再执行上面的命令。

最后执行
>   $ make check
> 
>   $ sudo make install
> 
>   $ which protoc
> 
>   $ protoc --version

还收一些软件可以提前安装：
> pip install --user Cython
> 
> pip install --user contextlib2
> 
> pip install --user jupyter
> 
> pip install --user matplotlib
> 
> pip install --user Cython
> 
> pip install --user contextlib2
> 
> pip install --user pillow
> 
> pip install --user lxml
> 
> pip install --user jupyter
> 
> pip install --user matplotlib

现在不安装也没关系，后面报错的时候会提示缺少什么软件再安装也行。

# 图片分类环境搭建
下载[https://github.com/tensorflow/hub/archive/master.zip](https://github.com/tensorflow/hub/archive/master.zip)到任意目录并解压。将该目录加入到PYTHONPATH环境变量中：
> export PYTHONPATH = $PYTHONPATH:$hub的目录$


> 注意：如果之前执行过pip install tensorflow-hub就不要再加环境变量了。

到 tensorflow_hub/pip_package/setup.py 所在的目录，执行
> python setup.py build
> 
> python setup.py install

# 下载样本训练

找一个目录，下载图片压缩包：

> curl -LO http://download.tensorflow.org/example_images/flower_photos.tgz
> 
>tar xzf flower_photos.tgz

到 examples/image_retraining/retrain.py 所在的目录，执行
> python retrain.py --image_dir /flower_photos

训练的时候会先下载模型，这里用的是inception模型，下载过程需要翻墙，因为模型是官网（谷歌）提供的。

训练好以后，会在/tem/ 目录下生成 .pb 文件和 output_labels.txt 文件。

# 通过安卓APP测试

下载官方提供的demo APP，对它进行改造：[https://github.com/tensorflow/examples/archive/master.zip](https://github.com/tensorflow/examples/archive/master.zip)。打开 lite/examples/image_classification 目录，用 Android Studio 打开 Android/app 目录。Android Studio会自动编译，编译成功后先安装apk到自己手机看一下。

下载格式转换文件[https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos/blob/master/convertor.py](https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos/blob/master/convertor.py)用于将.pb文件转为tflite文件，tflite才能给安卓使用。

- graph_def_file改成上面的pb路径
- input_arrays改成Placeholder
- output_arrays改成final_result

执行convertor.py很快就生成converted_model.tflite文件。把生成的tflite文件和上面的output_labels.txt文件一起复制到 image_classification/android/app/src/main/assets/ 下面。找到 ClassifierFloatMobileNet.java 文件，复制一份并改名为 ClassifierFloatInception.java。修改其中image size XY 都是299。修改modelPath和LabelPath为上面刚复制过去的文件。修改ClassifierActivity.java 中的classifier变量初始化为new ClassifierFloatInception(this)。

> inception模型的向量大小为299，见[https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos/blob/master/InceptionV4.pdf](https://github.com/davelet/tensorflow_transfer_learning_for_android_on_macos/blob/master/InceptionV4.pdf)

再次打包安装到手机上可以用手机识别不同花朵的图片试一下。只能识别菊花、蒲公英、向日葵、郁金香、玫瑰。