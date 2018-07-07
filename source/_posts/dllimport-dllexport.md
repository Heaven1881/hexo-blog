---
title: C/C++中的dllimport和dllexport
date: 2018-06-16 21:06:06
tags:
categories:
---


这两个关键字都是在使用DLL动态库时使用的（Windows ）

<!-- more -->

在编译静态库时，不需要使用这两个关键字。

## dllexport
在编译动态库时，使用`__declspec(dllexport)`来指定要导出的接口。没有被导出的函数，将无法通过DLL被外部程序调用，编译时会提示找不到符号。

> 除了`dllexport`，还可使用模块定义(.def)文件声明，(.def)文件为链接器提供了有关被链接程序的导出、属性及其他方面的信息。
 
## dllimport
对于外部程序来说，大部分情况下`dllimport`关键字不是必须的。不过使用`dllimport`可以帮助编译器生成更好的代码。可以通过下面这个例子来进步一了解：

假设`func`是DLL中的一个函数，并且导入的头文件中没有`__declspec(dllimport)`的定义，我们在另一个程序的main函数中尝试调用DLL中的函数。

```cpp
void func()

int main()
{
	func();
}
```

**编译器**将产生类似这样的调用

```x86asm
call func
```

然后，**链接器**把该调用翻译为类似这样的代码：

```x86asm
call 0x40000001       // ox40000001是"func"的地址
```

并且，**链接器**将产生一个**Thunk**，形如：

```x86asm
0x40000001: jmp DWORD PTR __imp_func
```

这里的`imp_func`是`func`函数在.exe的导入地址表中的函数槽的地址。然后，**加载器**只需要在加载时更新.exe的导入地址表即可。

如果使用了`__declspec(dllimport)`显示地导入函数，那么**链接器**就不会产生**Thunk**（如果不被要求的话），而直接产生一个间接调用。因此，下面的代码：

```cpp
__declspec(dllimport) void func1(void);

void main(void) 
{
    func1();
}
```

将产生如下调用：

```x86asm
call DWORD PTR __imp_func1
```

 因此，显示地导入函数能有效减少目标代码（因为不产生**Thunk**）。另外，在DLL中使用DLL外的函数也可以这样做，从而提高空间和时间效率。

## 使用dllimport导入DLL中的全局变量
一般来说，我们可以使用`extern`关键字来导出库中的全局变量，不过如果我们使用的是DLL，则最好用` __declspec(dllimport)`来代替`extern`，显式的告诉编译器这个变量来自于DLL，可以有助于生成更高效的代码。

关键字`__declspec(dllimport)`的作用主要体现在导出类的静态成员方面，可以使用如下方式导出类的静态成员：


```cpp
class __declspec(dllimport) Test
{  
public:  
    Test();  
    ~Test();  

    int myValue;  
    static int GetTotalValue()  
    {  
        return totalValue;  
    }  
private:  
    static int totalValue;  
};
```

这样，就可以直接在DLL外部使用静态成员函数`Test::GetTotalValue()`了

## 参考资料
> [declspec(dllexport) & declspec(dllimport) - cnblogs](http://www.cnblogs.com/xd502djj/archive/2010/09/21/1832493.html)
> [总结一下__declspec(dllimport)的作用 - cnblogs](https://blog.csdn.net/clever101/article/details/5421782)