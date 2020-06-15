---
layout: post
title: 使用pytorch进行图片分类器的训练
categories: [dev]
tags: [java]
---

在之前的文章《[通过tensorflow进行图片分类识别训练](/tfpicdetec/)》中我们通过google tensorflow 进行了图片分类的迁移学习。这里我们使用另一个流行的机器学习库pytorch进行训练。

> 机器学习让python大放异彩，让人诧异

使用pytorch比使用tensorflow更简单。之前学习tensorflow的时候，感觉入门门槛还是不低，现在看pytorch就觉得好多了。

# 图片的准备

这里使用前面tensorflow提供的图片训练样本集，可以从[http://download.tensorflow.org/example_images/flower_photos.tgz](http://download.tensorflow.org/example_images/flower_photos.tgz)下载。在项目里创建data目录（当然名称随便你），把5种花的文件夹都放进去。

# 图片的加载

pytorch 加载图片是把样本都放入loader里面。这里先定义一个方法将图片分为训练集合测试集：

```python
import torch
import torchvision
from torch import optim
from torchvision import models
from torchvision.transforms import transforms
import torch.nn as nn
import numpy as np
from matplotlib import pyplot as plt
from torch.utils.data.sampler import SubsetRandomSampler
from datetime import datetime

PATH = './data'


def load_both_train_test(data_dir, valid_size=0.2):
    train_transform = transforms.Compose([transforms.Resize(298), transforms.ToTensor(), ])
    test_transform = transforms.Compose([transforms.Resize(298), transforms.ToTensor()])
    train_data = torchvision.datasets.ImageFolder(data_dir, transform=train_transform)
    test_data = torchvision.datasets.ImageFolder(data_dir, transform=test_transform)
    num_train = len(train_data)
    index = list(range(num_train))
    split = int(np.floor(valid_size * num_train))
    np.random.shuffle(index)
    train_inx = index[split:]
    test_inx = index[:split]
    train_sampler = SubsetRandomSampler(train_inx)
    test_sampler = SubsetRandomSampler(test_inx)
    train_loader = torch.utils.data.DataLoader(dataset=train_data, sampler=train_sampler, batch_size=64)
    test_loader = torch.utils.data.DataLoader(dataset=test_data, sampler=test_sampler, batch_size=64)
    return train_loader, test_loader
```

下面开始训练，并隔一定间隔使用测试集测试当时的模型：

```python
if __name__ == '__main__':
    model = models.resnet50(pretrained=True)
    for p in model.parameters():
        p.requires_grad = False

    model.fc = nn.Sequential(nn.Linear(2048, 512), nn.ReLU(), nn.Dropout(0.2), nn.Linear(512, 10), nn.LogSoftmax(dim=1))
    criterion = nn.NLLLoss()
    optimizer = optim.Adam(model.fc.parameters(), lr=0.003)

    epochs = 1
    step = 0
    running_loss = 0
    print_every = 10
    train_losses, test_losses = [], []

    train_loader, test_loader = load_both_train_test(PATH)
    print("length are ", len(train_loader), len(test_loader))
    for e in range(epochs):
        for inputs, labels in train_loader:
            step += 1
            optimizer.zero_grad()
            logps = model.forward(inputs)
            loss = criterion(logps, labels)
            loss.backward()
            optimizer.step()
            running_loss += loss.item()
            print('step ===', step)
            if step % print_every == 9:
                test_loss = 0
                accuracy = 0
                model.eval()
                with torch.no_grad():
                    for inputs, labels in test_loader:
                        logps = model.forward(inputs)
                        batch_loss = criterion(logps, labels)
                        test_loss += batch_loss.item()
                        ps = torch.exp(logps)
                        top_p, top_class = ps.topk(1, dim=1)
                        equals = top_class == labels.view(*top_class.shape)
                        accuracy += torch.mean(equals.type(torch.FloatTensor)).item()

                train_losses.append(running_loss / len(train_loader))
                test_losses.append(test_loss / len(test_loader))
                print(f"Epoch {e + 1}/{epochs}.. "
                      f"Train loss: {running_loss / print_every:.3f}.. "
                      f"Test loss: {test_loss / len(test_loader):.3f}.. "
                      f"Test accuracy: {accuracy / len(test_loader):.3f}")
                running_loss = 0
                model.train()
```
每10步测试并打印精度结果，类似于：

```
Epoch 1/1.. Train loss: 3.388.. Test loss: 2.234.. Test accuracy: 0.387
```

# 模型的保存

为了以后方便使用或者继续训练，上面得到的model可以保存起来：
```python
    time = datetime.now().__str__().replace(' ', '_')
    torch.save({'state_dict': model.state_dict()}, './aerialmodel' + time + '.pth')
```
state_dict是模型的参数，也可以把模型图全保存起来，save的第一个参数直接写model就行。

# 代码
[https://github.com/davelet/pytorch-img-classification/blob/master/main.py](https://github.com/davelet/pytorch-img-classification/blob/master/main.py)
