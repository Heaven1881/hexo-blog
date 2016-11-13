---
title: gdb教程(2)：控制程序的执行
tags:
  - gdb
  - c/c++
  - step
  - next
categories: gdb教程系列
date: 2016-11-09 21:00:00
---


gdb可以说等价于你的程序的解释器，本文将介绍如何使用gdb查看你的程序执行状态。

<!-- more -->

# 相关文章
> [gdb教程(1)：如何开始][gdb-categories]
> gdb教程(2)：控制程序的执行
> [gdb教程(3)：断点(breakpoint)和观察点(watchpoint)][gdb-categories]
> [gdb教程(4)：函数调用栈][gdb-categories]

[gdb-categories]: /categories/gdb教程系列/

通过上一篇教程，你已经知道如何使用gdb来开始你的调试。现在，我们来介绍如何开始真正的调试。通过组合键`CTRL+C`，你可以随时让你的程序暂停运行，你还可以使用**断点(breakpoint)**来让程序在任意一行暂停，当程序暂停时，你不仅可以查看当前正在运行的代码，还可以查看并修改每一个变量的值。

# 暂停/继续 Debug程序

使用组合键`CTRL+C`来发送`SIGINT`信号来通知gdb暂停正在运行的程序。

```sh
(gdb) run
Starting Program: /home/ug/ryansc/a.out

Program received signal SIGINT, Interrupt.
0x80483b4 in main(argc=1, argv=0xbffffda4) at loop.c:5
5   while(1){
...
(gdb)
```

当程序处于暂停状态时，你可以使用`continue`命令来让程序继续执行，gdb对部分常用的命令支持简写，在这里我们可以使用简写命令`c`代替`continue`命令，两者的效果是一样的。

> **注意**：gdb几乎所有的命令都只能在程序暂停状态下输入。

# 查看程序当前执行位置
使用`list`命令(简写为`l`)来查看程序当前正在执行的代码，`list`命令会打印出当前执行代码附近的代码。例如下面的例子，当前代码执行到第8行，`list`命令会打印第8行附近的代码。

```sh
$ (gdb) list
3       int main(int argc, char **argv)
4       {
5         int x = 30;
6         int y = 10;
7       
8         x = y;
9       
10        return 0;
11      }

```

# 按行执行代码
使用`next`命令(简写为`n`)或着`step`命令(简写为`s`)来一行一行地执行你的代码。

```sh
5   while(1){
(gdb) next
7   }
(gdb)
```

**注意**：`step`和`next`是不同的，对于有函数调用的行，使用`step`命令会进入函数内部，而使用`next`命令则会直接跳过当前函数执行下一行。看下面的例子：

使用`next`命令时，函数`fun1()`被跳过。

```sh
(gdb)
11     fun1();
(gdb) next
12 }
```

使用`step`命令时，gdb进入函数`fun1()`内部。

```sh
(gdb)
11     fun1();
(gdb) step;
fun1 () at loop.c:5
5    return 0;
(gdb)
```

# 查看变量
使用`print`命令(简写为`p`)来打印当前作用域中的变量。假设你有两个变量`int n`和`char* s`，可以使用`print`命令来查看变量的具体内容。gdb会根据变量类型来确定输出格式。

```sh
(gdb) print n
$1 = 900
(gdb) print s
$3 = 0x8048470 "Hello World!\n"
(gdb)
```

> 注意：所有`print`返回都是`$### = {value}`格式的，`$###`中的数字是gdb用来记录你所检查的每一个变量的。

# 修改变量
使用`set`命令修改变量，例如，我们想要把`int n`的值改为3。

```sh
(gdb) set n = 3
(gdb) print n
$4 = 3
```

> 在一部分版本的gdb中，你需要使用`set var`命令代替`set`命令，例如`set var n = 3`。

# 调用函数
使用`call`命令，你可以调用几乎程序中的任何函数，包括系统库的函数和你自己实现的函数，例如：

```
(gdb) call abort()
```

# 从当前函数退出
当我们使用`step`命令进入一个函数后，可以使用`finish`命令让程序自动运行直到函数退出。这个命令还会告诉你函数的返回值(如果有)。`finish`命令的简写为`fin`

```
(gdb) finish
Run till exit from #0  fun1 () at test.c:5
main (argc=1, argv=0xbffffaf4) at test.c:17
17        return 0;
Value returned is $1 = 1

```

本节到这里就结束了，下一节我会介绍gdb中的**断点(breakpoint)**以及**观察点(watchpoint)**相关的操作。

# 参考资料
> [RMS's gdb Tutorial: How do I watch the execution of my program?](http://www.unknownroad.com/rtfm/gdbtut/gdbstep.html)
> [Continuing and Stepping - Debugging with GDB](https://sourceware.org/gdb/onlinedocs/gdb/Continuing-and-Stepping.html)
> [Debugging with GDB - List](ftp://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_46.html)

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。