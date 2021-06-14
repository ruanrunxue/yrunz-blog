---
title: 彻底弄懂Java的移位操作符
date: 2019-12-22
categories:
 - Java
tags:
 - java
author: 元闰子
---

## 前言

对于**移位操作符**，很多人既感到熟悉，又感到陌生。熟悉是因为移位操作符是最基本的操作符之一，几乎每种编程语言都包含这一操作符；陌生是因为除非是追求极致性能等罕见场景，否则也很难用得上它。打开JDK源码，你会发现移位操作符的身影极为常见，弄清楚它的用法，对阅读源码很有帮助。

移位操作是把数据看作是二进制数，然后将其向左或向右移动若干位的运算。在Java编程语言中，移位操作符包含三种，分别是 `<<`（左移）、 `>>`（带符号右移）和 `>>>`（无符号右移），这三种操作符都只能作用于`long`、`int`、`short`、`byte`、`char`这四种基本的整型类型上。

## 左移操作符 <<

左移操作符 `<<` 是将数据转换成二进制数后，**向左移若干位，高位丢弃，低位补零**。看如下例子：

```Java
public static void main(String[] args) {
    int i = -1;
    System.out.println("Before << , i's value is " + i);
    System.out.println("i's binary string is " + Integer.toBinaryString(i));
    i <<= 10;
    System.out.println("After << , i's value is " + i);
    System.out.println("i's binary string is " + Integer.toBinaryString(i));
}
```

Java的`int`占32位，因此对`i = -1`转换成二进制数，然后左移10位，其结果是左边高10位丢弃，右边低10位补`0`，再转换为十进制，得到`i = -1024`的结果。

![int -1 << 10 = -1024](https://tva1.sinaimg.cn/large/006tNbRwly1ga5hmfaa9kj317e0fgx6q.jpg)

因此，上述例子的输出结果为：

```
Before << , i's value is -1
i's binary string is 11111111111111111111111111111111
After << , i's value is -1024
i's binary string is 11111111111111111111110000000000
```

## 带符号右移操作符 >>

众所周知，Java中整型表示负数时，最高位为符号位，正数为`0`，负数为`1`。`>>` 是带符号的右移操作符，将数据转换成二进制数后，**向右移若干位，高位补符号位，低位丢弃**。对于正数作右移操作时，具体体现为高位补`0`；负数则补`1`。看如下例子：

```Java
public static void main(String[] args) {
    // 对正数进行右移操作
    int i1 = 4992;
    System.out.println("Before >> , i1's value is " + i1);
    System.out.println("i1's binary string is " + Integer.toBinaryString(i1));
    i1 >>= 10;
    System.out.println("After >> , i1's value is " + i1);
    System.out.println("i1's binary string is " + Integer.toBinaryString(i1));
    // 对负数进行右移操作
    int i2 = -4992;
    System.out.println("Before >> , i2's value is " + i2);
    System.out.println("i2's binary string is " + Integer.toBinaryString(i2));
    i2 >>= 10;
    System.out.println("After >> , i2's value is " + i2);
    System.out.println("i2's binary string is " + Integer.toBinaryString(i2));
}
```

例子中，`i1 = 4992`转换成二进制数，右移10位，其结果是左边高10位补`0`，右边低10位丢弃，再转换为十进制，得到`i1 = 4`的结果。同理，`i2 = -4992`，右移10位，左边高10位补`1`，右边低10位丢弃，得到`i2 = -5`的结果。

![int 4992 >> 10 = 4 和 int -4992 >> 10 = -5](https://tva1.sinaimg.cn/large/006tNbRwly1ga5j0f98r1j317v0u0qvb.jpg)

因此，上述例子的输出结果为：

```
Before >> , i1's value is 4992
i1's binary string is 1001110000000
After >> , i1's value is 4
i1's binary string is 100
Before >> , i2's value is -4992
i2's binary string is 11111111111111111110110010000000
After >> , i2's value is -5
i2's binary string is 11111111111111111111111111111011
```

## 无符号右移操作符 >>>

无符号右移操作符 `>>>`与`>>`类似，都是将数据转换为二进制数后右移若干位，不同之处在于，不论负数与否，结果都是**高位补零，低位丢弃**。看如下例子：

```Java
public static void main(String[] args) {
    int i3 = -4992;
    System.out.println("Before >>> , i3's value is " + i3);
    System.out.println("i3's binary string is " + Integer.toBinaryString(i3));
    i3 >>>= 10;
    System.out.println("After >>> , i3's value is " + i3);
    System.out.println("i3's binary string is " + Integer.toBinaryString(i3));
}
```

同样对`i3 = -4992`进行操作，转换成二进制数后，右移10位，其结果为左边高10位补`0`，右边低10位丢弃，再转换成十进制，得到`i3 = 4194299`的结果。

![int -4992 >>> 10 = 4194299](https://tva1.sinaimg.cn/large/006tNbRwly1ga5jpv18dzj31bi0huqv7.jpg)

因此，上述例子的输出结果为：

```
Before >>> , i3's value is -4992
i3's binary string is 11111111111111111110110010000000
After >>> , i3's value is 4194299
i3's binary string is 1111111111111111111011
```

## 真的懂了吗？

### 对 short、byte、char 的移位操作

再看如下例子：

```Java
public static void main(String[] args) {
    byte b = -1;
    System.out.println("Before >> , b's value is " + b);
    System.out.println("b's binary string is " + Integer.toBinaryString(b));
    b >>>= 6;
    System.out.println("After >> , b's value is " + b);
    System.out.println("b's binary string is " + Integer.toBinaryString(b));
}
```

Java的`byte`占8位，按照前面讲述的原理，对`b = -1`转换为二进制数后，右移6位，左边高6位补`0`，右边低位丢弃，其结果应该是`b = 3`。

![int -1 >>> 6 = 3 ?](https://tva1.sinaimg.cn/large/006tNbRwly1ga5kik778aj31bq0gee83.jpg)

真的这样吗？我们看一下例子运行的结果：

```
Before >> , b's value is -1
b's binary string is 11111111111111111111111111111111
After >> , b's value is -1
b's binary string is 11111111111111111111111111111111
```

*运行结果与我们预期的结果不对！*

原来，Java在处理`byte`、`short`、`char`的移位操作前，会**先将其转型成`int`类型，然后在进行操作**！特别地，当对这三者使用`<<=`、`>>=`和`>>>=`时，其实是得到对移位后的`int`进行低位截断后的结果！对例子改动一下进行验证：

```Java
public static void main(String[] args) {
    byte b = -1;
    System.out.println("Before >> , b's value is " + b);
    System.out.println("b's binary string is " + Integer.toBinaryString(b));
    System.out.println("After >> , b's value is " + (b >>> 6));
    System.out.println("b's binary string is " + Integer.toBinaryString(b >>> 6));
}
```

在该例子中，没有使用 `>>>=` 对 `b` 进行再赋值，而是直接将 `b >>> 6` 进行输出（需要注意的是，`b >>> 6`  的结果为 `int` 类型），其输出如下：

```
Before >> , b's value is -1
b's binary string is 11111111111111111111111111111111
After >> , b's value is 67108863
b's binary string is 11111111111111111111111111
```

因此，第一个例子中实际的运算过程应该是这样：

![byte -1 >>> 6 = -1](https://tva1.sinaimg.cn/large/006tNbRwly1ga5l8h7ogxj31go0rkhdy.jpg)

对于`short`和`char`的移位操作原理也一样，读者可以自行进行实验验证。

### 如果移位的位数超过数值所占有的位数会怎样？

到目前为止的所有的例子中，移位的位数都在数值所占有的位数之内，比如对`int`类型的移位都没有超过32。那么如果对`int`类型移位超过32位会怎样？且看如下例子：

```Java
public static void main(String[] args) {
    int i4 = -1;
    System.out.println("Before >>> , i4's value is " + i4);
    System.out.println("i4's binary string is " + Integer.toBinaryString(i4));
    System.out.println("After >>> 31 , i4's value is " + (i4 >>> 31));
    System.out.println("i4's binary string is " + Integer.toBinaryString(i4 >>> 31));
    System.out.println("After >>> 32 , i4's value is " + (i4 >>> 32));
    System.out.println("i4's binary string is " + Integer.toBinaryString(i4 >>> 32));
    System.out.println("After >>> 33 , i4's value is " + (i4 >>> 33));
    System.out.println("i4's binary string is " + Integer.toBinaryString(i4 >>> 33));
}
```

根据前面讲述的原理，对于`i4 >>> 31`我们很容易得出结果为`1`。

![int -1 >>> 31 = 1](https://tva1.sinaimg.cn/large/006tNbRwly1ga5m7iq40rj31gc0hcx6r.jpg)

*那么，`i4 >>> 32`的结果会是`0`吗？*

**NO**！Java对移位操作符的右操作数`rhs`有特别的处理，对于`int`类型，只取其低5位，也就是取`rhs % 32`的结果；对于long类型，只取其低6位，也即是取`rhs % 64`的结果。因此，对于`i4 >>> 32`，实际上是`i4 >>> (32 % 32) `，也即`i4 >>> 0 `，结果仍然是`-1`。

![int -1 >>> 32 = -1](https://tva1.sinaimg.cn/large/006tNbRwly1ga5mjmiy5jj31eq0hmx6r.jpg)

同理，对于`i4 >>> 33`等同于`i4 >>> 1`，其结果为`2147483647`。

![int -1 >>> 33 = 2147483647](https://tva1.sinaimg.cn/large/006tNbRwly1ga5mnx7mh2j31hq0hi4qs.jpg)

因此，上述例子的输出结果如下：

```
Before >>> , i4's value is -1
i4's binary string is 11111111111111111111111111111111
After >>> 31 , i4's value is 1
i4's binary string is 1
After >>> 32 , i4's value is -1
i4's binary string is 11111111111111111111111111111111
After >>> 33 , i4's value is 2147483647
i4's binary string is 1111111111111111111111111111111
```

对于`long`类型也是同样的道理，读者可以自行进行实验验证。

## 总结

移位操作符虽说是Java中最基本的操作符之一，但是若不彻底弄清楚其中细节，稍有不慎，便容易犯错。移位操作符实际上支持的类型只有`int`和`long`，编译器在对`short`、`byte`、`char`类型进行移位前，都会将其转换为`int`类型再操作。移位操作符在JDK源码中，最常见的用法便是将其当成是乘 `*` 或者除 `/` 操作符使用 ：对一个整型左移一位，相当于乘以2；右移一位，相当于除以2。其中原因就是，相比于使用 `*` 和 `/` ，在Java代码里使用 `<<` 和 `>>` 转换成的指令码运行起来会更高效些。
