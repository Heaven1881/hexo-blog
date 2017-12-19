---
title: 在跨平台环境使用结构体字节位(bit-fields)的注意事项
tags:
  - 跨平台
  - bit-fields
  - c/c++
categories:
  - 编程语言
date: 2017-12-18 21:58:49
---


考虑到网络流量以及运行效率，在涉及到网络同步时，大部分项目都会选择二进制协议。对于C/C++，虽然在Struct中使用bit-fields可以在一定程度上减少数据大小，但是也会带来一定的风险，需要根据实际情况来评估。

<!-- more -->

# 字节位(bit-fields)

字节位可以参考以下示例

```cpp
struct BitFields {
    int flag_1 : 1;
    int flag_2 : 1;
    int attrib_1 : 8;
    int attrib_2 : 8;
} __attribute__((__packed__))
```

上面的例子中，我们使用一个整型变量来保存2个整型变量以及2个不大于255的数字。和单独的变量比起来，这种方法可以缩减结构体所占用的空间。但是如果在跨平台的环境下，这样的写法就有问题。因为在不同的编译环境下这个结构体的大小是不一样的

在Linux环境，使用gcc编译并测试，结果为：

```cpp
sizeof(BitFields) = 3
```

在Windows环境，使用mingw编译并测试，结果为：

```cpp
sizeof(BitFields) = 4
```

这是因为不同编译器对bit-fields的补齐方式不一样导致的。结构体BitFields实际使用了18个bit的空间，因此最少需要使用3个字节(24个bit)来保存数据。那么为什么在Window平台下结构体的大小为4字节呢？这是因为Window的补齐长度取决于字段前的声明类型，在这里声明的类型为`int`(32个bit)，足以承担全部的数据，所以mingw在编译时直接分配了4个字节。

从这个结果来看，bit-field确实不怎么适合在跨平台编程下使用，但是如果还是想要在跨平台使用bit-field怎么办呢？有一种取巧的办法。

既然我们知道Windows和Linux下对bit-fields的补足方式的区别，那我们就主动构造一个在Window和Linus下都大小都相等的结构就好了。

```cpp
struct BitFields {
    int flag_1 : 1;
    int flag_2 : 1;
    int attrib_1 : 8;
    int attrib_2 : 8;
    int reverse : 14;
} __attribute__((__packed__))
```

上面的结构额外声明了14个数据位，现在，无论是Window还是Linux，数据大小都相等了。不过这种方法其实已经对平台有了限制，不能再称为跨平台了。

# C标准文档中关于bit-fields的说明
在C标准文档的附录J中，介绍了一些和bit-fields相关的未定义行为。

> **J.3 Implementation-defined behavior**
> A conforming implementation is required to document its choice of behavior in each of the areas listed in this subclause. The following are implementation-defined:
> **J.3.9 Structures, unions, enumerations, and bit-fields**
> - Whether a "plain" int bit-field is treated as a signed int bit-field or as an unsigned int bit-field (6.7.2, 6.7.2.1).
> - Allowable bit-field types other than _Bool, signed int, and unsigned int (6.7.2.1).
> - Whether atomic types are permitted for bit-fields (6.7.2.1).
> - Whether a bit-field can straddle a storage-unit boundary (6.7.2.1).
> - The order of allocation of bit-fields within a unit (6.7.2.1).
> - The alignment of non-bit-field members of structures (6.7.2.1). This should present no problem unless binary data written by one implementation is read by another.
> - The integer type compatible with each enumerated type (6.7.2.2).

总的来说有几个重要点：
- bit-fields的补齐方式是由编译器的实现决定的
- 同一个结构的bit-field字段的顺序也由编译器的实现决定

# 结论
总得来说，在跨平台编程中并不适合使用bit-fields。最稳妥的方法是还是手动进行数据的序列化操作。

# 参考资料
> [Size of struct with bitfields different between Linux (gcc) and Windows (mingw32 gcc)
](https://stackoverflow.com/questions/31349819/size-of-struct-with-bitfields-different-between-linux-gcc-and-windows-mingw32)