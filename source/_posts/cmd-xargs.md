---
title: xargs命令学习笔记
tags:
  - xargs
  - linux
categories:
  - 系统命令
date: 2016-05-21 17:14:04
---


`xargs`是一个分批执行命令的工具，通过`xargs`可以实现很多快捷的功能。

<!-- more -->

# 前言

前段时间在服务器上遇到一个问题：在一个目录及其子目录下包含若干文件，要求找到文件内容同时包含两个指定字符串的文件。其实`grep`可以提供类似的功能，但是`grep`命令只能处理两个字符在同一行的情况，和我们的要求不太一样。后来在论坛里找到了下面这条命令

```bash
$ grep -l key_1 *.txt | xargs grep -l key_2
```

这条命令在满足通配符`*.txt`的所有文件中，寻找同时包含字符串`key_1`和`key_2`的文件。

这个例子足以展示`xargs`的能力，接下来我们就详细介绍`xargs`命令。

# xargs命令

这篇笔记的学习内容大多数参考自[这里](http://www.softpanorama.org/Tools/xargs.shtml)，还有一部分内容是根据我自己实际测试得出的理解，如果有错误欢迎大家指正。

`xargs`的主要功能就是分批执行命令，它可以以`stdin`或者管道作为输入。`xargs`的主要工作流程是这样的：

1. 从`stdin`或者管道中读取输入
2. 将输入的字符串数据分割为字符串列表，默认情况下，以空白符(空格，换行符和制表符)作为分隔符
3. 将获取的字符串列表使用空格连接起来，直接拼接到对应的命令后面
4. 执行步骤3中产生的命令

例如，我们的`test.txt`文件内容如下

```
=== test.txt ===
ta tb tc
td te
tf
```

输入如下命令

```bash
$ cat test.txt | xargs echo
```

等价于执行以下命令

```bash
$ echo ta tb tc td te tf
```

# 分组执行命令

`xargs`允许对输入的字符串列表进行分组，使用`-n`选项可以指定分组大小，根据分组的数量，`xargs`会多次执行给定命令，借助这个参数可以做到减少每一条命令的长度，以下命令

```bash
$ cat test.txt | xargs -n 4 ech
```

等价于依次执行如下多条命令

```bash
$ echo ta tb tc td
$ echo te tf
```

在上面的情况，我们指定分组大小为4，但是总共只有6个参数，因此第二条命令只有2个参数。默认情况下，即使最后参数不足`xargs`也会继续执行命令，使用`-x`选项让`xargs`在剩余参数不足时停止执行。

命令

```bash
$ cat test.txt | xargs -n 4 -x echo
```

等价于:

```bash
$ echo ta tb tc td
```

# 调试命令

`xargs`还提供了许多选项，下面两个选项主要用于调试：
 
- `-t`：在执行每条命令的时候将命令内容打印出来
- `-p`：在执行每条命令前询问用户

# 指定替换位置

`xargs`会简单的将获取的参数拼接到命令的后面，如果我们希望指定插入参数的位置，可以使用`-I`选项指定替换的字符串

```bash
$ cat test.txt | xargs -I {} echo hello! {}, have a nice day!
```

等价于:

```bash
$ echo hello! ta tb tc td te tf, have a nice day!
```

比较遗憾的是，这种方法只能设置一个替换位置，类似下面命令的使用方法`xargs`目前不支持。(当然也有有可能是我的学习不够透彻)

```bash
$ cat test.txt | xargs -I {} echo hello {name}, welcome to {address}!
```

**PS**：其实可以使用`awk`和`xargs`配合管道达到类似的效果。

# 参考资料

> [xargs Command Tutorial](http://www.softpanorama.org/Tools/xargs.shtml)