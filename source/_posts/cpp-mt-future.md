---
title: C++11多线程(5)：Future
tags:
  - C++11
  - future
  - promise
categories: C++11多线程
date: 2018-10-03 11:44:19
---


C++11对多线程进行的支持，让我们可以直接跳过系统API直接在语言层面编写多线程程序，直接的好处就是程序的可移植性得到了很大的提高。这篇文章主要介绍**future**头文件中的相关内容。

<!-- more -->

> *正在进行C++11多线程编程相关的学习吗？[点击这里](/categories/C-11%E5%A4%9A%E7%BA%BF%E7%A8%8B/)查看这个分类的更多文章。*


头文件future里主要包含多线程中用于异步访问变量的工具。例如，我们可以将耗时的计算放到其他线程，然后在未来的某个时间点访问计算结果，从而避免主线程的阻塞问题。借助`future`类和`promise`类，我们可以很轻松完成这些功能。

## 概念
在future头文件中有两个主要概念：`Providers`和`Futures`，`Providers`是数据的提供方，`Futures`是数据的使用方。例如我们需要在线程`A`中进行运算，然后将结果传递给线程`B`，那么在这里，线程`A`就是`Providers`，线程`B`就是`Futures`。

> 本文中涉及到一些重复的单词，为避免混淆，通过大小写来区分。例如`Futures`表示的是概念，`future`或者`std::future`表示具体的C++类型

![](/uploads/providers_and_futures.jpg)

`Providers`和`Futures`一般是成对出现的，每一对`Providers`和`Futures`实际包含同一块数据区域，我们称作`shared-state`，即共享数据状态。`shared-state`主要保存以下内容：
- `Providers`需要传递给`Futures`的数据，`Providers`可以向`Futures`传递数据，也可以向`Futures`抛出一个异常。
- `shared-state`本身的状态，例如未初始化，或者是就绪（`ready`）状态

## std::promise 和 std::future
在头文件`<future>`中，最常用的类就是`std::promise`和`std::future`。在这里`std::promise`对象就是概念里提到的`Providers`，而`std::future`就是`Futures`。

我们先来看下面这个示例：

```cpp
#include <iostream>       // std::cout
#include <functional>     // std::ref
#include <thread>         // std::thread
#include <future>         // std::promise, std::future

void print_int(std::future<int>& fut) {
  int x = fut.get();
  std::cout << "value: " << x << '\n';
}

int main()
{
    std::promise<int> prom;                      // create promise

    std::future<int> fut = prom.get_future();    // engagement with future

    std::thread th1(print_int, std::ref(fut));  // send future to new thread

    prom.set_value(10);                         // fulfill promise
                                                // (synchronizes with getting the future)
    th1.join();
    return 0;
}
```

上面的代码涉及到两个线程，我们不妨成为主线程和副线程：
- 主线程，主要执行`main`函数
- 副线程，主要执行`print_int`函数，负责打印主线程传递过来的整型变量

主线程首先创建了`std::promise`对象，然后通过`get_future`方法获取与之对应的`std::future`对象。两个对象共享同一份`shared-state`数据。

```cpp
std::future<int> fut = prom.get_future();
```

副线程拿到`std::future`对象，通过`get`函数来取得对应的结果。不过此时`shared-state`并不处于`ready`的状态，因此副线程在`get`方法内阻塞等待。

``` cpp
int x = fut.get();
```

主线程通过调用`prom`的`set_value`方法向`shared-state`中写入数据，并将其状态设置为`ready`，此时副线程从`get`方法中返回，获取到主线程设置的值（在代码中，这个值为10）。

```cpp
prom.set_value(10); 
```

看完了基础示例，我们来详细介绍每个`Providers`类型和`Futures`类型。

## Providers
`Providers`类型主要包括`std::promise`和`std::packaged_task`。`std::promise`我们在上面已经简单介绍过，而`std::packaged_task`本质上是对`std::promise`的封装，`packaged_task`可以让我快速的将一个普通的函数封装为一个支持`promise`和`future`类型的函数，而不需要修改原来的函数。
  
接下来，我们会依次介绍`std::promise`和`std::packaged_task`。

### promise
**主要构造函数**

| 构造函数 | 说明 | 
| -------- | ------- | 
| `promise()` | 默认构造函数，构造一个空对象 |
| `promise (const promise&) = delete` | 拷贝构造函数（删除） |
| `promise (promise&& x) noexcept` | 移动构造函数 |

和C++11其他多线程类型一样，`promise`对象允许移动构造(move)，不允许拷贝构造(copy)。

**promise::get_future**
```cpp
future<T> get_future();
```
获取对应的`future`对象，该`future`对象与当前`promise`对象指向同一个`shared-state`。因此，每个`promise`对象只能生成一个`future`对象，第二次调用`get_future()`函数会触发`std::future_error::future_already_retrieved`的异常。

一旦对一个`promise`对象使用了`get_future()`方法，那么这个`promise`对象就应该在未来某一个时刻“履行承诺”。可以有如下选择：
- 调用`set_value()`方法来向`Shared-state`写入数据
- 调用`set_exception()`方法来向`Shared-state`抛出异常，`future`中可以取得该异常。

一个没有按约定“履行承诺”的`promise`对象会在析构时向`Shared-state`抛出`future_error`类异常，相关的错误信息是`broken_promise`（无法履行的承诺）。

**promise::set_value**
```cpp
void set_value (const T& val);
void set_value (T&& val);
```
向`Shared-state`中写入变量`val`，并将`Shared-state`设置为`ready`。

如果对应的`future`对象此刻正在`get`函数中等待结果，那么函数会继续运行并返回变量`val`。

一个模板中比较特殊的例子，如果`promise`使用`void`来实例化模板，则`set_value`函数只会将`Shared-state`设置为`ready`，不会写入任何值。

`set_value`的使用可以本文开头的示例，这里不再重复。

**promise::set_exception**
```cpp
void set_exception (exception_ptr p);
```
向`Shared-state`中写入异常，并将`Shared-state`设置为`ready`。

如果对应的`future`对象此刻正在`get`函数中等待结果，那么函数会返回并抛出异常。

```cpp
#include <iostream>       // std::cin, std::cout, std::ios
#include <functional>     // std::ref
#include <thread>         // std::thread
#include <future>         // std::promise, std::future
#include <exception>      // std::exception, std::current_exception

void get_int (std::promise<int>& prom) {
  int x;
  std::cout << "Please, enter an integer value: ";
  std::cin.exceptions (std::ios::failbit);   // throw on failbit
  try {
    std::cin >> x;                           // sets failbit if input is not int
    prom.set_value(x);
  }
  catch (std::exception&) {
    prom.set_exception(std::current_exception());
  }
}

void print_int (std::future<int>& fut) {
  try {
    int x = fut.get();
    std::cout << "value: " << x << '\n';
  }
  catch (std::exception& e) {
    std::cout << "[exception caught: " << e.what() << "]\n";
  }
}

int main ()
{
  std::promise<int> prom;
  std::future<int> fut = prom.get_future();

  std::thread th1 (print_int, std::ref(fut));
  std::thread th2 (get_int, std::ref(prom));

  th1.join();
  th2.join();
  return 0;
}
```
**promise::set_value_at_thread_exit**
```cpp
void set_value_at_thread_exit (const T& val);
void set_value_at_thread_exit (T&& val);
```

将变量`val`保存到`Shared-state`中，但是并不立即将`Shared-state`的状态设置为`ready`，而是在当前线程退出后（在当前线程中所有储存资源都被销毁后）将其状态设置为`ready`。

需要注意的是，对`promise`对象调用函数`set_value_at_thread_exit`后，`Shared-state`中就已经写入了具体的数值，在这之后如果继续调用`set_value`，会触发异常，异常类型为`promise_already_satisfied`。

### packaged_task
`packaged_task`可以让我快速的将一个普通的函数封装为一个支持`promise`和`future`类型的函数，而不需要修改原来的函数。参考下面的代码：

```cpp
#include <iostream>     // std::cout
#include <future>       // std::packaged_task, std::future
#include <chrono>       // std::chrono::seconds
#include <thread>       // std::thread, std::this_thread::sleep_for

// count down taking a second for each value:
int countdown (int from, int to) {
  for (int i=from; i!=to; --i) {
    std::cout << i << '\n';
    std::this_thread::sleep_for(std::chrono::seconds(1));
  }
  std::cout << "Lift off!\n";
  return from-to;
}

int main ()
{
  std::packaged_task<int(int,int)> tsk (countdown);   // set up packaged_task
  std::future<int> ret = tsk.get_future();            // get future

  std::thread th (std::move(tsk),10,0);   // spawn thread to count down from 10 to 0

  // ...

  int value = ret.get();                  // wait for the task to finish and get result

  std::cout << "The countdown lasted for " << value << " seconds.\n";

  th.join();

  return 0;
}
```

代码首先利用`countdown`函数构建了`packaged_task`对象，使之能够返回`promise`对象，然后将对应的函数托管到线程中执行，然后通过`future::get`来获取其结果。

除了可以用`std::move`将`packaged_task`对象中的函数托管到线程中运行，也可以直接调用`operator()`函数来在当前线程中执行对应的函数。
```cpp
std::thread th(std::move(tsk), 10, 0); // 托管到其他线程中运行

tsk(10, 0); // 在当前线程中运行
```
**主要构造函数**

| 构造函数 | 说明 | 
| -------- | ------- | 
| `packaged_task() noexcept` | 默认构造函数，构造一个空对象 |
| `template <class Fn>`<br>`explicit packaged_task (Fn&& fn)` | 将函数`fn`封装为一个返回`promise`对象的函数 |
| `packaged_task (packaged_task&) = delete` | 拷贝构造函数（删除） |
| `packaged_task (packaged_task&& x) noexcept` | 移动构造函数 |


**packaged_task::make_ready_at_thread_exit**
调用`packaged_task`中的函数，将结果保存到对应的`Shared-state`中，但是延迟到当前线程退出后再将`Shared-state`中的状态设置为`ready`。

**packaged_task::reset**
保持`packaged_task`中设置的函数不变，但是分配一个新的`Shared-state`对象。旧的`Shared-state`对象(如果有)会被废弃，其效果等同于对原`packaged_task`调用了析构函数。

## Futures
Future类型主要包括`future`和`shared_future`。`shared_future`是`future`的补充，借助`shared_future`，我们可以在多个地方访问`Shared-state`

### future
`future`对象用来从某一类`Providers`(例如`promise`和`packaged_task`)中获取数据，如果对应的`Provider`处于不同的线程，`future`对象可以自动的处理线程同步的问题。

当我们说一个`future`对象是有效的(valid)，也就是说该`future`对象和某一个`Shared-state`是关联的(associated)。一般来说，从以下途径获得的`future`对象是有效的：
- `std::async`
- `std::promise::get_future`
- `std::packaged_task::get_future`

**主要构造函数**

| 构造函数 | 说明 | 
| -------- | ------- | 
| `future() noexcept` | 默认构造函数，构造一个空对象 |
| `future (const future&) = delete` | 拷贝构造函数（删除） |
| `future (future&& x) noexcept` | 移动构造函数 |

对于程序来说，无效的`future`对象是无用的。使用默认构造函数产生的`future`对象是无效的。

`future`对象允许移动构造(move)，不允许拷贝构造(copy)，

**future::get**
如果当前`future`对象关联的`Shared-state`中状态为`ready`，那么返回其中保存的数值（或者异常）。

如果当前`future`对象关联的`Shared-state`中状态不为`ready`，那么当前线程会进入阻塞状态，直到`Shared-state`变为`ready`。

一旦`get`函数成功返回（或者正确抛出异常），`future`对象会解除和当前`Shared-state`的关联，在此之后，当前`future`对象不再有效（valid）。因此对于每一个`Shared-state`，`get`函数只能调用一次。

**future::valid**
返回当前`future`是否有效。简单来说只有与`Shared-state`关联的`future`是有效的。

```cpp
#include <iostream>       // std::cout
#include <future>         // std::async, std::future
#include <utility>        // std::move

int get_value() { return 10; }

int main ()
{
  std::future<int> foo,bar;
  foo = std::async (get_value);
  bar = std::move(foo);

  if (foo.valid())
    std::cout << "foo's value: " << foo.get() << '\n';
  else
    std::cout << "foo is not valid\n";

  if (bar.valid())
    std::cout << "bar's value: " << bar.get() << '\n';
  else
    std::cout << "bar is not valid\n";

  return 0;
}
```
输出：
```
foo is not valid
bar's value: 10
```

**future::share**
获取一个`shared_future`对象，该对象引用相同的`Shared-state`。与`future`不同的是，`shared_future`可以多次调用`get`函数，并且允许被拷贝，可以同时由多个线程持有。

需要注意的是，在调用`share`函数获取对应的`shared_future`后，原始的`future`会解除和`Shared-state`的关联，因此不再有效。

```cpp
#include <iostream>       // std::cout
#include <future>         // std::async, std::future, std::shared_future

int get_value() { return 10; }

int main ()
{
  std::future<int> fut = std::async (get_value);
  std::shared_future<int> shfut = fut.share();

  // shared futures can be accessed multiple times:
  std::cout << "value: " << shfut.get() << '\n';
  std::cout << "its double: " << shfut.get()*2 << '\n';

  return 0;
}
```
输出：
```
value: 10
its double: 20
```

**future::wait**
```cpp
void wait() const;
```
阻塞等待，直到`Shared-state`变为`ready`。

**future::wait_for**
```cpp
template <class Rep, class Period>
  future_status wait_for (const chrono::duration<Rep,Period>& rel_time) const;
```
阻塞等待，直到`Shared-state`变为`ready`，或者达到超时时间。

```cpp
#include <iostream>       // std::cout
#include <future>         // std::async, std::future
#include <chrono>         // std::chrono::milliseconds

// a non-optimized way of checking for prime numbers:
bool is_prime (int x) {
  for (int i=2; i<x; ++i) if (x%i==0) return false;
  return true;
}

int main ()
{
  // call function asynchronously:
  std::future<bool> fut = std::async (is_prime,700020007); 

  // do something while waiting for function to set future:
  std::cout << "checking, please wait";
  std::chrono::milliseconds span (100);
  while (fut.wait_for(span)==std::future_status::timeout)
    std::cout << '.';

  bool x = fut.get();

  std::cout << "\n700020007 " << (x?"is":"is not") << " prime.\n";

  return 0;
}
```
输出：
```
checking, please wait..........................................
700020007 is prime.
```

## 参考资料
> [Future - C++ Reference](http://www.cplusplus.com/reference/future/)