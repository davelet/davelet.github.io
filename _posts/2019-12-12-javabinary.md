---
layout: post
title: Java中数的二进制存储
categories: [dev]
tags: [java]
---

其实我们都知道整数在计算机中是用二进制表示的，并且也知道各种语言都提供了位运算相关的能力。这篇文章也没什么新意，如果你对二进制很熟悉，请跳过去看下一篇文章；不过二进制平时使用的不多，所以你还是可以在这里回顾一下自己的知识。

# 整数的二进制表示

这个估计大家都很熟悉，比如2表示成二进制就是10，10表示成二进制就是1010。不过看见这个问题，大多数人脑子里出现的都是正数的，其实我们还有负数呢！

十进制的负数表示是在前面加一个负号-，例如-123。二进制使用最高位表示符号位，用1表示负数，用0表示正数：这我们都知道。哪个是最高位呢？整数有4种类型byte、short、int、long，分别占1、2、4、8个字节，即分别占8、16、32、64位，每种类型的符号位都是其最左边的一位。

为方便举例，下面假定类型是byte，即从右到左的第8位表示符号位。

- byte a=-1，将最高位变为1，二进制应该是10000001
- byte a=-127，将最高位变为1，二进制应该是11111111

但实际上恰恰相反，-1的二进制是11111111，-127的表示时10000001。大家说：“补码嘛！”没错，这个是补码，前面的叫原码。补码就是原码取反后加1：

- -1：1 的原码是00000001，取反是11111110，加1变成11111111
- -2：2 的原码是00000010，取反是11111101，加1变成11111110
- -127：127 原码是011111111，取反是10000000，加1变成10000001

已经知道一个负数的二进制补码，要转化成正数，不用先减去1再取反，也是先取反再加1，一样的求补就行：

- -110的二部制补码是10010010，取反是01101101，加1是01101110，对应十进制是2+4+8+32+64=110
- 如果先减再反则10010010-1=10010001，取反01101110，结果一样

补码的这个特性让我们使用起来很方便。当然还得想一下：为啥用补码呢？难道用原码不可以吗？

> 另一个问题就是正数取反加1后高位一定是1吗？难道不会是0吗？要使得加1后高位是0只能是高位进位了，这样取反的结果就只能是11111111，也就是原码是00000000，补码也是00000000。原码是0，它的负数当然也是0。（这里也说明了不能先减再反，因为0没法先减，如果强制借位就和其他补码又一样了）
> 
> 还有一个问题是补码是10000000的数对应正数是哪个？10000000取反加1的结果还是10000000（减1取反的结果也是10000000），一个数跟它相反数相等说明这个数是0

负数采用补码的原因是只有这样计算机才能方便（正确）的进行加减法。计算机智能做加法，减法就是加上对应的负数：1-1就是1+（-1），如果用原码就是
```bash
 1 -> 00000001
-1 -> 10000001
+ ------------
-2 -> 10000010
```
结果竟然是-2。而用补码表示：
```bash
 1 -> 00000001
-1 -> 11111111
+ ------------
 0 -> 00000000
```
结果正确。再比如5-3：
```
 5 -> 00000101
-3 -> 11111101
+ ------------
 2 -> 00000010
```
结果也正确。看上去很难理解，使用上却完全正确，奇妙不？

### 溢出
从上面的过程可以看出来，当两个正数相加导致高位变成1会使计算机将结果认为成负数：
```
 127 -> 01111111
   1 -> 00000001
+   ------------
-128 -> 10000000
```

## 十六进制
十六进制使用0-9和A-F，这大家都知道。这里强化记忆一下：将16进制数赋值给变量前面要加0x：
```
int a = 0x7B; // 就是十进制123
```
从Java 7 开始可以将二进制串赋值给变量了，前面加0B即可：
```
int a = 0b11001
```

# 移位
二进制可以移位。移位操作分三种：

1. 左移 `<<`，高位舍弃，低位补0。相当于乘2
2. 无符号右移 `>>>`，右边舍弃，左边补0
3. 有符号右移 `>>`，右边舍弃，左边补高位数字，相当于除以2