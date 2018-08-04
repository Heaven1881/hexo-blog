---
title: C++11多线程(1)：Thread
tags:
  - C++11
  - thread
categories: C++11多线程
date: 2018-07-28 08:09:35
---


C++11对多线程进行的支持，我们可以直接跳过系统API直接在语言层面编写多线程程序，直接的好处就是多线程程序的可移植性得到了很大的提高。这篇文章主要介绍`std::thread`。

<!-- more -->

> *正在进行C++11多线程编程相关的学习吗？[点击这里](/categories/C-11%E5%A4%9A%E7%BA%BF%E7%A8%8B/)查看这个分类的更多文章。*

## 构造
`std::thread`的默认构造函数只会产生一个空对象，不会创建具体线程。我们可以通过指定函数名和相关的参数来创建一个线程。


```cpp
void foo() { std::cout << "in foo()" << std::endl; }

void bar(int n) { std::cout << "in bar(): " << n << std::endl; }

int main()
{
	std::thread first; // 空对象
	std::thread second(bar, 5); // 执行函数bar()，参数n=5
	first = std::thread(foo); // 执行函数foo()

	return 0; 
}

```

如果我们希望在线程中运行类的成员函数，我们需要同时提供类的对象。

```cpp
class A
{
public:
	void run()
	{
		std::thread thr(&A::runInThread, this, 5); // 调用该成员的RunInThread函数，参数为5
		thr.join();
	}

	void runInThread(int n)
	{
		std::cout << "In Thread:" << n << std::endl;
	}
}

int main()
{
	A a;
	a.run();
}

```

## 赋值
我们可以使用`swap()`方法来交换两个`thread`对象的内容

```cpp
std::thread thr1(foo);
std::thread thr2(bar, 2);
thr1.swap(thr2); // 两者的内容交换
```
`thread`不允许拷贝构造(copy)，但是可以移动构造(move)。所以我们可以使用`thread`的右值构造函数将一个`thread`对象的内容转移给另一个空对象。
```cpp
std::thread thr1(foo);

// std::thread thr2 = thr1; // 错误！不允许拷贝构造
std::thread thr2 = std::move(thr1); // 正确，移动构造，thr1最终变为空对象
```

## 释放
我们可以通过成员函数`join()`和`detach()`来释放`thread`对象对线程的控制权。
- `join()`： 等待线程函数执行完毕后再释放线程控制权
- `detach()`：直接释放线程控制权，线程仍然可以继续执行

如果一个`thread`对象不为空，那么在析构时必须调用`join()`或者`detach()`来释放线程的控制权。

不允许对空的`thread`对象调用`join()`或`detach()`方法，通常我们可以调用`joinable()`方法来判断是否可以调用`join()`和`detach()`。一个`thread`对象的`joinable()`方法返回`true`当且仅当这个`thread`对象控制着一个具体的线程。在以下情况时，`thread`对象不是`joinable`的：
- `thread`对象是通过默认构造函数的空对象
- `thread`对象曾经通过`move()`方法将自己的线程控制权转移给其他`thread`对象
- `thread`对象曾经调用过`join()`或者`detach()`方法。



## 参考资料
> [thread - C++ Refrence](http://www.cplusplus.com/reference/thread/thread/)