---
layout: post
title: Java类文件简析
categories: [dev]
tags: [java, asm]
---
java类文件的结构实际很简单，它不像本地可执行文件那样，类文件保留了源代码中的结构和几乎全部的符号信息。

1. 有一块用于描述修饰符、名称、父类、接口、类上注解
2. 有一块用于描述所有字段，包括字段修饰符、名称、类型、其上注解。
3. 有一块用于描述所有方法，包括构造方法，但不包括父类方法。描述内容也是修饰符、方法名、返回类型、参数类型、方法注解。不过这里包含了方法体，也就是编译后的字节码指令。

源文件和类文件不同点是：
1. 类文件只描述一个类，而源文件可以包含多个类。一个类如果包含内部类编译后会生成两个类文件，主类文件包含指向内部类的引用，内部类会定义一个方法指向父类。
2. 类文件不包含注释信息，但可以包含类、字段、方法、代码的属性用来关联这些元素额外的信息。jdk5开始用注解代替这个功能，所以现在属性没什么用了。
3. 类文件不包含包和引入部分，所以所有类型都是全限定名。

另外类文件包含一个constant pool 块，它是一个数组，包括了类中的所有数值、字符串、类型常量。它们定义后，用到的时候直接通过下标访问。ASM 隐藏了这个区域的细节。

Java虚拟机规范中说的类结构如下：
![类结构](/images/post/klass.png)

# 内部名
许多情况下只有类和接口，其他的类型就不复存在。一个类继承的父类、实现的接口、抛出的异常，只可能是类或接口，不可能是基本类型或数组。类文件中使用到的类和接口都是它们的全限定名，称为“内部名”。内部名是把全限定名中的点改成斜线，比如String的内部名是java/lang/String。

# 类型描述符
只有类和接口会用到内部名，字段类型之类的用到的叫类型描述符。基本类型都用一个字母表示：boolean是Z，char是C，byte是B，short是S，int是I，float是F，long是J，double是D。引用类型的表示是L加上内部名加分号，比如String的描述符是Ljava/lang/String;。数组的描述符是维数个左方括号加上元素类型的内部名，比如int[]是[I，Object[][]是[[Ljava/lang/Object;。

|  类型   | 描述符  | 说明 |
|  ----  | ----   | ----|
| boolean  | Z | 布尔
| char  | C | 字符
| byte  | B | 字节
| short  | S | 短整型
| int  | I | 整型
| float  |  F| 单精度浮点
| long  |  J| 长整型
| double  | D | 双精度浮点
| Object  | Ljava/lang/Object; | 引用类型
| int[]  | [I | 一维数组
| Object[][]  | [[Ljava/lang/Object; | 二维数组

# 方法描述符
方法描述符是一个字符串，内容是类型描述符的列表，这些类型描述符分别是描述参数和返回值的。方法描述符的组成是一个小括号，里面是参数的类型描述符的直接拼接（不包含任何其他符号），后面跟上返回值的类型描述符，如果没有返回值就跟上大写的V，表示void。方法描述符不包含方法名称或参数名称的信息。

下面是一个方法描述符的例子：

|  源代码中的方法申明         | 方法描述符  |
|  :----------------       | :-------  | 
| void m(int i, float f)    | (IF)V | 
| int m(Object o)          | (Ljava/lang/Object;)I| 
| int[] m(int i, String s) | (ILjava/lang/String;)[I | 
| Object m(int[] i)        | ([I)Ljava/lang/Object; | 

方法描述符很简单，只要你理解了类型描述符是怎么回事。
