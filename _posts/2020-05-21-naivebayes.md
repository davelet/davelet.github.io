---
layout: post
title: 朴素贝叶斯方法入门
categories: [dev]
tags: [ide]
---

大家都知道贝叶斯定理。朴素贝叶斯就是使用贝叶斯定理进行分类的方法。为什么叫“朴素”呢？因为它简单，英文叫“萌蠢”(naive)，会假设个体特征相互独立。不过简单不代表它效果差，在不少分类领域，朴素贝叶斯方法带来的性价比高到惊人。本文通过两个例子简单使用一下朴素贝叶斯方法（NB， naive Bayes）。

# 贝叶斯定理
非数学专业大一的三门数学课分别是高数、概率、线代。

> 我们上学，不管你读到多高学历，全日制教育都要一直包含三个学科：数学、英语、哲学

贝叶斯定理很早就出现在概率课程中了，因为它简单。说到条件概率、先验概率等名词，大家应该还都有印象。贝叶斯定理是说条件概率P(Y|X)的曲线求解思路：
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?P(Y|X)=\frac{P(X|Y)\times&space;P(Y)}{P(X))}">
</div>
意思就是说，要求X发生的条件下Y发生的概率，可以通过计算Y发生的条件下X发生的概率与Y发生概率的积，除以X发生的概率。前提是后面几个概率更加容易求出来。

# 病人分类案例
这是网上一个比较简单但是典型的案例。假设今天医院收到6个门诊病人，他们的情况如下：

| 针状   | 职业     | 疾病   |
| ------ | -------- | ------ |
| 打喷嚏 | 护士     | 感冒   |
| 打喷嚏 | 农民     | 过敏   |
| 头疼   | 建筑工人 | 脑震荡 |
| 头疼   | 建筑工人 | 感冒   |
| 打喷嚏 | 教师     | 感冒   |
| 头疼   | 教师     | 脑震荡 |

---
现在又来了第七个病人，是一个打喷嚏的建筑工人。那么他是因为感冒的概率有多大？也就是求P(感冒|打喷嚏×建筑工人)，已知是打喷嚏的建筑工人，那么他感冒的概率。

为了演示，这里数据量很少；实际生活中这么少的数据量是完全得不出结论的。

我们用A表示打喷嚏，B代表建筑工，C代表感冒，直接套用公式：
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?P(C|A\times&space;B)&space;=\frac{P(A\times&space;B|C)\times&space;P(C)}{P(A\times&space;B)}" title="P(C|A\times B)\\ =\frac{P(A\times B|C)\times P(C)}{P(A\times B)}" />
</div>

因为打喷嚏和建筑工人是相互独立的，所以
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?{P(A\times&space;B)}&space;=&space;P(A)\times&space;P(B)" title="{P(A\times B)} = P(A)\times P(B)" />
</div>

于是得到
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?P(C|A\times&space;B)\\&space;\\=\frac{P(A\times&space;B|C)\times&space;P(C)}{P(A\times&space;B)}&space;\\&space;=&space;\frac{P(A|C)\times&space;P(B|C)\times&space;P(C)}{P(A)\times&space;P(B)}" title="P(C|A\times B)\\ \\=\frac{P(A\times B|C)\times P(C)}{P(A\times B)} \\ = \frac{P(A|C)\times P(B|C)\times P(C)}{P(A)\times P(B)}" />
</div>

后面这几个概率都可以通过样本直接得到：
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?P(C|A\times&space;B)\\&space;\\=\frac{P(A\times&space;B|C)\times&space;P(C)}{P(A\times&space;B)}&space;\\&space;=&space;\frac{P(A|C)\times&space;P(B|C)\times&space;P(C)}{P(A)\times&space;P(B)}&space;\\&space;=&space;\frac{0.66&space;\times&space;0.33&space;\times&space;0.5}{0.5\times&space;0.33}\\=0.66" title="P(C|A\times B)\\ \\=\frac{P(A\times B|C)\times P(C)}{P(A\times B)} \\ = \frac{P(A|C)\times P(B|C)\times P(C)}{P(A)\times P(B)} \\ = \frac{0.66 \times 0.33 \times 0.5}{0.5\times 0.33}\\=0.66" />
</div>

因此，结论是这个打喷嚏的建筑工人有66%的可能是感冒。

如果直接看计算过程，这个66%的值是通过
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?\frac{P(A|C)\times&space;P(B|C)\times&space;P(C)}{P(A)\times&space;P(B)}=0.66" title="\frac{P(A|C)\times P(B|C)\times P(C)}{P(A)\times P(B)}=0.66" />
</div>
得到的，也就是感冒中打喷嚏的比例乘以感冒中建筑工人的比例再乘以感冒的比例，除以打喷嚏的建筑工人的比例得到的。很难理解，但这就是数学。

> 公式编辑由[codecogs](https://www.codecogs.com/latex/eqneditor.php?lang=zh-cn)支持

上面我们认为打喷嚏和建筑工人是相互独立的，这就是朴素贝叶斯和常规贝叶斯的差别：贝叶斯是不能简单认为特征相互独立的，而朴素贝叶斯要求各特征相互独立。

假设个体有F1,F2,...,Fn个特征且相互独立，要看它们是C1,C2,...,Cm这些类别中的哪一类，方法就是通过贝叶斯定理求概率最大的那一类（这就叫分类器）
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?max_{i}\{P(C_{i}|F_{1}F_{2}...F_{n})\}=\frac{P(F_{1}F_{2}...F_{n}|C_{i})\times&space;P(C_{i})}{P(F_{1}F_{2}...F_{n})}" title="max_{i}\{P(C_{i}|F_{1}F_{2}...F_{n})\}=\frac{P(F_{1}F_{2}...F_{n}|C_{i})\times P(C_{i})}{P(F_{1}F_{2}...F_{n})}" />
</div>
因为分母是类别无关的“常量”，既然是比较大小就可以忽略。又因为是独立特征，所以
<div align="center">
<img src="https://latex.codecogs.com/gif.latex?P(C_{i}|F_{1}F_{2}...F_{n})=\frac{P(F_{1}F_{2}...F_{n}|C_{i})\times&space;P(C_{i})}{P(F_{1}F_{2}...F_{n})}&space;\propto&space;P(F_{1}F_{2}...F_{n}|C_{i})\times&space;P(C_{i})\\=P(F_{1}|C)P(F_{2}|C)...P(F_{n}|C)P(C)" title="P(C_{i}|F_{1}F_{2}...F_{n})=\frac{P(F_{1}F_{2}...F_{n}|C_{i})\times P(C_{i})}{P(F_{1}F_{2}...F_{n})} \propto P(F_{1}F_{2}...F_{n}|C_{i})\times P(C_{i})\\=P(F_{1}|C)P(F_{2}|C)...P(F_{n}|C)P(C)" />
</div>

式子最后边的每一项都可以通过样本统计到，所以结果很容易出来。

> 所以朴素贝叶斯的分类公式就是类别做为各个特征的条件，求各个特征条件概率的积最后乘以分类的概率

# 垃圾邮件识别
现在进阶一点，看网上一个对邮件进行分类的例子，识别垃圾邮件和普通邮件。使用朴素贝叶斯分类器的目标值就是P(垃圾邮件∣具有某特征)。假设我们有大量样本已经，然后需要判断这个邮件：
```
我司可办理正规发票（保真）17%增值税发票点数优惠
```
