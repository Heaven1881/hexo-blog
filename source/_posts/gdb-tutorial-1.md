---
title: gdb教程(1)：如何使用gdb
tags:
 - gdb
 - c
 - c++
 - debuger
categories: gdb相关
---

> 参考资料：http://www.unknownroad.com/rtfm/gdbtut/gdbuse.html

<!-- more -->

在使用gdb之前，你需要让编译器编译出可以让gdb使用的可执行程序。也就是在编译程序的时候，使用`-g`选项让编译器生成一些调试需要使用的的信息。`-g`选项是必须的，否则gdb无法获取一些必要的信息，例如符号表。一般情况，我们称这种方法编译出来的程序为Debug版本。

# 编译Debug程序
在编译时加上选项`-g`来在生成gdb需要的调试信息

```sh
$ gcc -g test.cpp -o test
```
> **注意**：如果你的程序包含多个源文件，需要使用`-g`编译

# 运行Debug程序
直接使用gdb命令来开始你的调试

```sh
$ gdb test
```
一旦gdb成功启动，你可以使用`run`命令让程序开始运行，如果你的程序需要通过命令参数来运行，你可以同时在这里传入对应的参数。

```sh
$ (gdb) run arg1 arg2 ...
```

# 重新运行Debug程序
你可以使用`kill`命令来终止一个程序的运行，然后再次调用`run`命令来重新运行你的Debug程序

```sh
$ (gdb) kill
Kill the program being debugged? (y or n) y
$ (gdb) run
```

# 退出调试
使用`quit`命令可以退出gdb调试

```sh
$ (gdb) quit
The program is running. Exit anyway? (y or n) y
```

# 查看gdb帮助
使用命令`help`查看gdb支持的命令，你可以使用`help xxx`的命令来进一步查看某一个命令的详细信息。

```
$ (gdb) help
List of classes of commands:

aliases -- Aliases of other commands
breakpoints -- Making program stop at certain points
data -- Examining data
files -- Specifying and examining files
internals -- Maintenance commands
obscure -- Obscure features
running -- Running the program
stack -- Examining the stack
status -- Status inquiries
support -- Support facilities
tracepoints -- Tracing of program execution without stopping the program
user-defined -- User-defined commands

Type "help" followed by a class name for a list of commands in that class.
Type "help" followed by command name for full documentation.
Command name abbreviations are allowed if unambiguous.
```



