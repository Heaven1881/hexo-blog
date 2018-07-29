---
title: C++11 智能指针
date: 2018-07-23 21:55:22
tags: C++11
categories: 编程语言
---


C++11中一共引入了三种智能指针。

<!-- more -->

## std::unique_ptr
管理独占资源的指针，在任何时间点，资源只能唯一地被一个`unique_ptr`占有。当`unique_ptr`离开作用域，所包含的资源被释放。

```cpp
std::unique_ptr<int> uptr(new int);
std::unique_ptr<int[]> uptra(new int[5]);
```

`std::unique_ptr`不提供复制语义（不允许拷贝构造和拷贝赋值），只支持移动语义（通过`std::move`移动）。当把`std::unique_ptr`赋给另外一个对象时，资源的所有权就会被转移。

```cpp
std::unique_ptr<int> uptr(new int);
std::unique_ptr<int> uptr2 = uptr; // 错误，不允许左值赋值
std::unique_ptr<int> uptr2 = std::move(uptr); // 正确，赋值后uptr指针不指向任何对象
```

## std::shared_ptr
管理共享的指针，允许多个拷贝。内部使用了引用计数的机制来保证其安全性，当最后一个`shtard_ptr`离开作用域时，内存才会自动释放。

```cpp
std::shared_ptr<int> sptr(new int(10));
std::shared_ptr<int> sptr2 = std::make_shared<int> (10);
std::shared_ptr<int> sptr3 = sptr2; // 引用计数增加
```

上面的代码，当`sptr2`赋值给`sptr3`后，两者都指向同一个对象，该对象的引用计数变为2。这样的计数被称为`强引用(strong reference)`，此外，还有一种引用计数叫做`弱引用(weak reference)`。我们接下来会介绍。

## std::weak_ptr
为什么会有`weak_ptr`这个类型呢，我们来看下面的这个`shared_ptr`循环引用的例子。

```cpp
class B;
class A
{
public:
	 A() : m_sptrB(nullptr) { };
	 ~A()
	 {
		  cout<<" A is destroyed"<<endl;
	 }
	 std::shared_ptr<B> m_sptrB;
};
class B
{
public:
	 B(  ) : m_sptrA(nullptr) { };
	 ~B( )
	 {
		  cout<<" B is destroyed"<<endl;
	 }
	 std::shared_ptr<A> m_sptrA;
};

int main( )
{
	 std::shared_ptr<B> sptrB( new B );
	 std::shared_ptr<A> sptrA( new A );
	 sptrB->m_sptrA = sptrA;
	 sptrA->m_sptrB = sptrB;
}
```

上面的例子中，对象A引用对象B，对象B又引用对象A。这样的结果会导致离开`main`函数后`sptrA`和`sptrB`的引用计数无法降为0，造成内存泄漏。

为了解决这个问题，C++11引入了`weak_ptr`，其拥有和`share_ptr`类似的共享语义，但是并不真正持有资源。`share_ptr`中的弱引用计数指的就是`weak_ptr`的引用次数。当强引用计数变为0时，`share_ptr`所持有的的资源就会被释放（即使当前弱引用计数不为0）。

`weak_ptr`并不真正持有资源，因此也不支持普通指针对应的`*`,`->`操作符。当需要使用`weak_ptr`指向的资源时，我们需要把`weak_ptr`转换为`shared_ptr`，可以使用`lock()`方法来完成对应的操作。

```cpp
std::shared_ptr<int> sptr(new int);
std::weak_ptr<int> wptr = sptr;

std::shared_ptr<int> wsptr = wptr.lock();
```

我们可以使用`weak_ptr`来解决上面的循环引用问题。

```cpp
class B;
class A
{
public:
	 A() : m_sptrB(nullptr) { };
	 ~A()
	 {
		  cout<<" A is destroyed"<<endl;
	 }
	 std::weak_ptr<B> m_sptrB;
};
class B
{
public:
	 B(  ) : m_sptrA(nullptr) { };
	 ~B( )
	 {
		  cout<<" B is destroyed"<<endl;
	 }
	 std::weak_ptr<A> m_sptrA;
};

int main( )
{
	 std::shared_ptr<B> sptrB( new B );
	 std::shared_ptr<A> sptrA( new A );
	 sptrB->m_sptrA = sptrA;
	 sptrA->m_sptrB = sptrB;
}
```

这里我们根据语义将类`A`和`B`对对方的引用替换为若引用，可以解决问题。实际上，只需要将`A`或者`B`中任意一个的指针替换为`weak_ptr`就可以解决循环引用的问题。

## 参考资料
> https://www.jianshu.com/p/e4919f1c3a28