---
title: gdb教程(4)： 函数调用栈
tags:
  - gdb
  - call stack
  - c/c++
categories: gdb教程系列
date: 2016-11-13 11:45:04
---


通过查看函数调用栈，我们可以从整体上了解程序执行的状态，并且更容易发现程序中的问题。本片文章介绍gdb关于函数调用栈的操作。

<!-- more -->

# 相关文章
> [gdb教程(1)：如何开始][gdb-categories]
> [gdb教程(2)：控制程序的执行][gdb-categories]
> [gdb教程(3)：断点(breakpoint)和观察点(watchpoint)][gdb-categories]
> gdb教程(4)：函数调用栈

[gdb-categories]: /categories/gdb教程系列/

# 查看函数调用栈
使用命令`backtrace`(简写`bt`)来查看函数调用栈

```
(gdb) backtrace
#0  func2 (x=30) at test.c:5
#1  0x80483e6 in func1 (a=30) at test.c:10
#2  0x8048414 in main (argc=1, argv=0xbffffaf4) at test.c:19
#3  0x40037f5c in __libc_start_main () from /lib/libc.so.6
(gdb) 
```

从上面的例子，我们可以发现：当前程序正在执行函数`func2()`，而函数`func2()`在函数`func1(a=30)`内被调用，相关的代码的位置在`test.c:19`。

# 切换函数栈帧
设想这么一个情况，对于上面提到的例子，我们发现`func2()`中的错误是由于`func1()`中传入了错误的参数导致的，现在希望可以看`func1()`中的错误现场。

对于这样的需求，可以使用gdb命令`frame`加上函数栈帧的编号来实现，函数栈帧的编号指的是每一个函数栈帧前面以`#`开头的编号了，例如我想要将当前栈帧从`func2()`切换到`func1()`，那么我只要执行如下命令。

```
(gdb) frame 2
#2  0x8048414 in main (argc=1, argv=0xbffffaf4) at test.c:19
19        x = func1(x);
(gdb) 
```

`frame 2`表示切换到编号为2的栈帧，现在我们已经成功切换到`func1()`栈帧中了，我们可以使用`print`或者`set`来查看或修改变量的值。

# 查看当前函数栈的情况

使用`info frame`来查看当前函数栈的基础信息

```
(gdb) info frame
Stack level 2, frame at 0xbffffa8c:
 eip = 0x8048414 in main (test.c:19); saved eip 0x40037f5c
 called by frame at 0xbffffac8, caller of frame at 0xbffffa5c
 source language c.
 Arglist at 0xbffffa8c, args: argc=1, argv=0xbffffaf4
 Locals at 0xbffffa8c, Previous frame's sp is 0x0
 Saved registers:
  ebp at 0xbffffa8c, eip at 0xbffffa90
```

使用`info locals`来查看当前函数内所有变量的值

```
(gdb) info locals
x = 30
s = 0x8048484 "Hello World!\n"
```

使用`info args`来查看当前函数的参数以及他们的值

```
(gdb) info args
argc = 1
argv = (char **) 0xbffffaf4
```

# 参考资料
> [RMS's gdb Tutorial: How do I use the call stack?](http://www.unknownroad.com/rtfm/gdbtut/gdbstack.html)

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
