---
title: c++的四种类型转换操作
tags:
  - c/c++
  - cast
  - 类型转换
categories: 编程语言
date: 2016-11-18 20:07:55
---


相对于传统的C的类型转换，C++提供了更丰富的类型转换。在这里简单介绍一下他们之间的区别

<!-- more -->

## const\_cast
const\_cast操作符仅仅用于显示转换使用`const`和`volatile`修饰的变量，这也是c++里唯一一个可以将常量转化成变量的操作符。需要保证转换前后的类型是一样的，例如`const char`只能转换为`char`，而不能转换为`int`。

```cpp
int main()
{
    const char cc = 'a';
    char c = const_cast<char>(cc);
    int i = const_cast<int>(cc);    // 编译器报错，无法将const char 转换为 int
}
```

## static\_cast
这是最常用的一种转换方式，用于：

- 基本数据类型之间的转换，例如`int`，`char`，`float`，`double`之间的转换
- `void*`和其他任意类型指针的转换
- 把空指针转换为任意其他类型的空指针
- 还可以进行子类和基类之间的指针转换，但是并不会检查类型转换是否安全。

```cpp
class A {}
class B : public A {}
class C {}

int main()
{
    int i = 123456;
    B b;
    C c;
    
    char c1 = i;                            // 可以编译通过，但是会有警告
    char c2 = static_cast<char>(i);         // OK
    
    double* pd = static_cast<double*>(&i);  // 错误：无关类型转换
    
    void* pvb = static_cast<void*>(&b);     // OK
    B* pb = static_cast<B*>(pvb);           // OK
    C* pc = static_cast<C*>(pvb);           // 这里可以编译通过，但是会产生运行错误
    
    B* pnil = static_cast<B*>(NULL);        // OK
    
    A* pa = &B;
    pb = static_cast<B*>(pa);               // OK
    pc = static_cast<C*>(pa);               // 可以编译通过，但是会产生运行错误
}

```

## dynamic\_cast
dynamic\_cast操作符功能与static\_cast类似，不过其主要用于指针和引用的转换。dynamic\_cast会检查转换是否成功。

- 尝试将基类指针转换为子类指针时，如果转换失败，则会返回NULL。
- 尝试将基类引用转换为子类引用时，如果转换失败，则会抛出bad_cast异常。
- 对于NULL指针与void指针，dynamic\_cast的功能与static\_cast相同。

```cpp
class A {}
class B : public A {}
class C {}
class D : public A {}

int main()
{
    B b;
    C c;
    D d;
    
    A* pa = &B;
    B* pb = dynamic_cast<B*>(pa);   // OK
    D* pd = dynamic_cast<D*>(pa);   // pd == NULL
    
    D* pd = dynamic_cast<D*>(&b);  // pc == NULL，因为B和C没有继承关系   
}
```

dynamic\_cast是唯一一个使用默认c风格转换无法实现的功能，这样的操作也会带来额外的开销。

## reinterpret\_cast
reinterpret\_cast相对于上面的两种转换来说有着更广泛的适用范围。它仅仅重新解释类型，并没有进行二进制数据的转换。reinterpret\_cast从某种意义上对编译器进行了欺骗，同时带来了一定的不安全性。使用这个类型转换时要慎重。适用的场合包括

- `int`和指针的转换
- 任意类型指针之间的转换
- 函数指针的转换

```cpp
class A {}
class B {}

int main()
{
    A* pa = new A();
    B* pb;
    
    pb = static_cast<B*>(pa);       // 编译通过，但是运行报错
    pb = dynamic_cast<B*>(pa);      // OK，实际运行返回NULL
    pb = reinterpret_cast<B*>(pa);  // OK，但是实际的结果无法预期
    
    delete pa;
}
```



---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
