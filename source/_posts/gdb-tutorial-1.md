---
title: gdb教程(1)：如何开始
date: 2016-11-09 12:00:00
tags:
 - gdb
 - c/c++
 - debug
 - fork
 - attach
categories: gdb教程系列
---

本文介绍如何使用gdb开始你的Linux/Unix调试之旅。

<!-- more -->

# 相关文章
> gdb教程(1)：如何开始
> [gdb教程(2)：控制程序的执行][gdb-categories]
> [gdb教程(3)：断点(breakpoint)和观察点(watchpoint)][gdb-categories]
> [gdb教程(4)：函数调用栈][gdb-categories]

[gdb-categories]: /categories/gdb教程系列/

# 编译Debug程序

在使用gdb之前，你需要让编译器编译出可以让gdb使用的可执行程序。也就是在编译程序的时候，使用`-g`选项让编译器生成一些调试需要使用的的信息。`-g`选项是必须的，否则gdb无法获取符号表等必要信息。一般情况，我们称这种方法编译出来的程序为Debug版本。

在编译时加上选项`-g`来在生成gdb需要的调试信息

```sh
$ gcc -g test.cpp -o test
```
> **注意**：如果你的项目包含多个源文件，需要保证每次编译时都是用了`-g`选项，否则gdb可能无法调试对应的文件。

# 运行Debug程序
直接使用gdb命令加上你的Debug程序路径作为参数来开始你的调试。

```sh
$ gdb test
```
一旦gdb成功启动，你可以使用`run`命令让程序开始运行，如果你的程序需要通过命令参数来运行，你可以同时在这里传入对应的参数。

```sh
$ (gdb) run arg1 arg2 ...
```

# 退出调试
使用`quit`命令可以退出gdb调试。

```sh
$ (gdb) quit
The program is running. Exit anyway? (y or n) y
```

# 调试运行中的进程
有时候我们需要对一个正在运行的进程进行调试，可以使用`attach`命令来让gdb进入到一个正在运行的程序。

```sh
$ (gdb) attach $PID
```

其中`$PID`指的是进程的ID(process id)，在Linux/Unix环境下，可以使用命令`ps -ax`来查看所有进程的信息。

> **注意：**需要确保被调试的进程也是使用`-g`选项编译过的。

你可以使用`detach`命令来退出正在调试的进程。

```sh
$ (gdb) detach
Detaching from program: xxxxx
```

# 调试fork()程序
有些程序会使用`fork()`或者`vfork()`函数来创建新的进程。在大多数系统下，gdb会继续进行当前进程的调试，而不是进入fork产生的子进程。如果我们希望在程序fork时让gdb进入子进程，那么我们可以在程序进入子进程的时候增加一个sleep调用，让你有足够的时间查询对应的进程ID，然后使用命令`attach`进入子进程。

当然在一些平台上，gdb已经对这个功能有了很好的支持，可以自动attach到fork产生的子进程中。在Linux平台上，这一功能需要保证内核版本为`2.5.46`或者更高。

如果你希望在程序fork时能进入子进程，可以使用如下命令设置对应的fork模式:
```
$ (gdb) set follow-fork-mode child
```

使用`show follow-fork-mode`来查看当前的“fork模式”，当然，如果你向同时调试父进程和子进程，可以使用如下命令来让gdb进入子进程时仍然保持和父进程的连接。

```sh
$ (gdb) set detach-on-fork off
```

关于调试fork()程序，更详细的信息可以参考文章底部的参考连接。

本节到这里就结束了，在下一节，我会介绍如果检查并控制程序的执行情况。

# 参考链接
> [RMS's gdb Tutorial: How do I use gdb?](http://www.unknownroad.com/rtfm/gdbtut/gdbuse.html)
> [Forks - Debugging with GDB](https://sourceware.org/gdb/onlinedocs/gdb/Forks.html)

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
