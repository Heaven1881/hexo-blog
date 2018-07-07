---
title: 'C++11特性： std::forward 和 std::move'
tags:
  - 左值
  - 右值
  - C++11
categories: 编程语言
date: 2018-06-23 07:41:43
---


std::forward和std::move是C++11引入的新函数，要介绍这两个函数，就需要从左值和右值开始说起

<!-- more -->

## 左值和右值
简单来说，如果一个变量有名字，那么他就是**左值(lvalue)**，否则就是**右值(rvalue)**。

**左值(lvalue)**既可以出现在赋值符号`=`的左边和右边，**右值(rvalue)**则只能出现在赋值符号`=`右边。
```
int a = get_num(); // 变量a是左值，函数get_num()的返回值为右值

// 下面的调用是非法的，右值不允许出现在赋值符号左边
get_num() = 1; 
```

右值的生命周期通常比较短，一般在右值所在表达式结束后就立即销毁
```
int a = 5 + get_num(); // get_num() 返回的值为右值，在完成变量a的赋值后则被销毁
```

可以使用`const`类型的引用来指向一个右值，此时将延长该右值的生命周期。
```cpp
const Test& t = getTest(); // 被引用后，右值不会立即销毁
t.func();
```

不能使用非`const`引用指向右值，下面的调用是非法的
```
Test& t = getTest(); // 错误，不能使用非const引用
```
在C++11中，可以使用如下函数声明来针对左值和右值进行函数区分
```
int testLRValue(const Test& test); // 允许传入左值和右值
int testLValue(Test& test); // 只允许传入左值
int testRValue(Test&& test); // 只允许右值，C++11新特性
```
既然用`const Test& test`的方法可以同时支持左值和右值，为什么还需要为左值和右值分别提供不同的函数呢？因为右值的声明周期很短，意味着在调用`testRValue(Test&& test)`函数后，变量`test`会被销毁。有时候可以通过这个性质来优化我们的代码，例如在拷贝构造函数中，我们可以直接将右值包含的资源进行转移（因为函数调用后右值本身会被销毁），而不需要将右值的资源拷贝过来。

## std::move
了解了左值和右值后，我们就可以讨论`std::move`的作用了。`std::move`的作用就是将一个左值转化为右值。类似于下面函数的声明：
```
T&& move(T& val)
```

## std::forward
之前提到`int testLRValue(const Test& test)`既可以传入左值，也可以传入右值，那么在函数内，变量`test`应该是左值还是右值呢？c++11 中对 rvalue 作了明确的定义：
> Things that are declared as rvalue reference can be lvalues or rvalues. The distinguishing criterion is: if it has a name, then it is an lvalue. Otherwise, it is an rvalue.

如果一个变量有名字，那么它就是**左值**，否则是**右值**，所以`test`应该是左值。我们可以通过`std::move`将变量`test`转为右值，不过仍然有局限性：

如果我们希望变量的引用状态不受函数结构的影响，即变量`test`保持上一层的逻辑结构：
- 如果调用方式为`testLRValue(getTest())`，那么我们希望函数内的变量是右值
- 如果调用方式为`testLRValue(test)`，那么我们希望函数内的变量是左值

这样的逻辑在C++11之前是无法实现的，C++11引入了`std::forward`函数，它接受一个参数，然后返回这个参数最初的引用。参考下面的使用 ：
```
TYPE* acquire_obj(ARG& arg)
{
   return new TYPE(std::forward<ARG&>(arg));
}
```

## 特例
关于右值引用，需要注意的是C++11虽然引入了右值符号`&&`，但是在函数声明中，`&&`**并不一定代表右值引用**，事实上仅仅对于具体的类型，`&&`才表示右值引用，例如：
```
class Test
{
	...
};

int getInt(Test&& t)
```
上面的类型`Test`是一个具体的类型，因此在函数`getInt`中，变量`t`是右值引用。但是对于推导类型，例如模板，`auto`等，`&&`是则可以同时表示左值和右值引用，例如：
```
template <typename T>
void func(T&& arg)
{
}
```
上面的代码中我们无法确定`arg`是左值引用还是右值引用，因为`T`是一个推导类型。如果我们传递左值给`func`，那么会得到`void func(T& arg)`，如果我们传递右值，那么就会得到`void func(T&& arg)`。



相关资料：
http://www.cnblogs.com/catch/p/3507883.html
http://www.cnblogs.com/catch/p/3500678.html