---
title: gdb教程(3)： 断点(breakpoint)和观察点(watchpoint)
tags:
  - gdb
  - break
  - watch
categories: gdb教程系列
date: 2016-11-10 20:37:34
---


在绝大多数情况下，我们并不需要从头开始一步一步的调试程序的。在这种情况下，**断点(breakpoint)**和**检查点(watchpoint)**就起到了很大的作用。

<!-- more -->

# 相关文章

> [gdb教程(1)：如何开始][gdb-categories]
> [gdb教程(2)：控制程序的执行][gdb-categories]
> gdb教程(3)：断点(breakpoint)和观察点(watchpoint)]
> [gdb教程(4)：函数调用栈][gdb-categories]

[gdb-categories]: /categories/gdb教程系列/

# 断点(breakpoint)
你可以通过设置断点来告诉gdb在程序执行到哪一行的时候暂停，你也可以将断点设置到某一函数上，这样程序会在调用这个函数时停止。然后你可以在这里检查变量的值或者查看程序的堆栈来找到程序中的问题。

## 将断点设置在某一行
使用`break`命令(简写为`b`)加上行号来在对应行设置断点

```sh
(gdb) break 19
Breakpoint 1 at 0x80483f8: file test.c, line 19
```

如果你的程序包括多个文件，你可以通过`文件名:行号`的形式来指定断点位置

```sh
(gdb) break test.c:19
Breakpoint 2 at 0x80483f8: file test.c, line 19
```
## 将断点设置在函数上
使用`break`命令加上对应的函数名来在让程序在调用函数时触发断点

```sh
(gdb) break func1
Breakpoint 3 at 0x80483ca: file test.c, line 10
```

如果你希望将断点设置在c++类的成员函数上，可以使用`类名::函数名`的形式设置断点

```sh
(gdb) break TestClass::testFunc(int) 
Breakpoint 1 at 0x80485b2: file cpptest.cpp, line 16.
```

## 管理断点
你可以使用`info breakpoints`命令来查看当前设置的所有断点

```sh
(gdb) info breakpoints
Num Type           Disp Enb Address    What
2   breakpoint     keep y   0x080483c3 in func2 at test.c:5
3   breakpoint     keep y   0x080483da in func1 at test.c:10
```

使用`disable`(`enable`)命令来禁用(启用)断点

```sh
(gdb) disable 2
(gdb) info breakpoints
Num Type           Disp Enb Address    What
2   breakpoint     keep n   0x080483c3 in func2 at test.c:5
3   breakpoint     keep y   0x080483da in func1 at test.c:10
```

使用`delete`命令来删除断点

```sh
(gdb) delete 2
(gdb) info breakpoints
Num Type           Disp Enb Address    What
3   breakpoint     keep y   0x080483da in func1 at test.c:10
```

# 观察点(watchpoint)
观察点与断点类似，他们之间不同的地方在于停止的逻辑，断点让程序在执行到代码的某一行时停止，而观察点让程序在某一变量发生改变时停止。

## 设置写观察点

```c
#include <stdio.h>

int main(int argc, char **argv)
{
  int x = 30;
  int y = 10;

  x = y;

  return 0;
}
```

以上面的代码为例，如果我们希望程序在变量`x`被修改时暂停，那么可以使用`watch`命令。

```sh
(gdb) watch x
Hardware watchpoint 4: x
(gdb) c
Continuing.
Hardware watchpoint 4: x

Old value = 30
New value = 10
main (argc=1, argv=0xbffffaf4) at test.c:10
10      return 0;
```

> **注意**：在使用`watch`命令观察变量`x`时，需要确保`x`处于当前的作用域内。

## 其他观察点
观察点除了可以监控变量的写操作，还可以监控读操作。使用`rwatch`来设置**读观察点**，使用`awatch`来设置**读写观察点**。

## 管理观察点
观察点也是断点的一种，因此断点支持的管理操作对观察点同样适用。使用`info breakpoints`可以查看当前观察点。

```sh
(gdb) info breakpoints
Num Type           Disp Enb Address    What
1   breakpoint     keep y   0x080483c6 in main at test.c:5
4   hw watchpoint  keep y   x
```

本节到这里就结束了，下一节我会介绍gdb关于函数堆栈的操作。

# 参考资料
> [RMS's gdb Tutorial: How do I use breakpoints?](http://www.unknownroad.com/rtfm/gdbtut/gdbbreak.html)
> [RMS's gdb Tutorial: How do I use watchpoints?](http://www.unknownroad.com/rtfm/gdbtut/gdbwatch.html)

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
