---
title: C++11多线程(2)：Mutex
tags:
  - C++11
  - mutex
categories: C++11多线程
date: 2018-08-04 09:10:32
---


C++11对多线程进行的支持，我们可以直接跳过系统API直接在语言层面编写多线程程序，直接的好处就是多线程程序的可移植性得到了很大的提高。这篇文章主要介绍`mutex`相关内容。

<!-- more -->

> *正在进行C++11多线程编程相关的学习吗？[点击这里](/categories/C-11%E5%A4%9A%E7%BA%BF%E7%A8%8B/)查看这个分类的更多文章。*

## Mutex
### std::mutex
这是最基本也是最轻量级的互斥量，提供的主要方法如下：
- `lock`：锁住当前互斥量，如果互斥量正在被其他线程占用，那么当前线程会阻塞等待，直到这个互斥量被释放为止。
- `try_lock`：尝试锁住当前互斥量，如果互斥量正在被其他线程占用，则函数会直接退出并返回`false`
- `unlock`：释放当前互斥量，让其他线程可以通过`lock`锁住当前互斥量

```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex

std::mutex mtx;           // mutex for critical section

void print_block (int n, char c) {
  // critical section (exclusive access to std::cout signaled by locking mtx):
  mtx.lock();
  for (int i=0; i<n; ++i) { std::cout << c; }
  std::cout << '\n';
  mtx.unlock();
}

int main ()
{
  std::thread th1 (print_block,50,'*');
  std::thread th2 (print_block,50,'$');

  th1.join();
  th2.join();

  return 0;

```

需要注意的是，当调用`lock()`时，如果当前`mutex`正在被同一个线程占用，那么`lock()`方法将会永远阻塞（死锁）。如果需要对一个`mutex`连续调用`lock()`方法，请使用`std::recursice_mutex`。

### std::recursive_mutex
`recursive_mutex`允许同一个线程重复调用`lock()`方法，内部维护了一个计数器，当`lock()`时计数器加一，`unlock()`时计数器减一，当引用计数为0时，其他线程可以调用`lock()`直接锁上当前互斥量。

### std::timed_mutex
与`recursive_mutex`一样，`timed_mutex`也是为了给原始`mutex`提供改进。我们上面提到`mutex`可以通过`try_lock()`来尝试锁住一个互斥量并且避免阻塞，实际使用时，大概是下面这种形式：

```cpp
std::mutex mtx;

void threadFunc()
{
	// do something before lock()
	while(mtx.try_lock());
	// do something after lock()
	mtx.unlock();
}
```

`timed_mutex` 提供了`try_lock_for()`和`try_lock_until()`函数，允许指定一段时间作为超时时间。如果当前互斥量正在被其他线程占用，那么函数会保持阻塞直到超时时间到达为止。

```cpp
std::timed_mutex mtx;

void threadFunc()
{
	// do something before lock()
	while(mtx.try_lock_for(std::chrono::milliseconds(100)));
	// do something after lock()
	mtx.unlock();
}
```

### std::recursive\_timed\_mutex
`recursive_timed_mutex`则是结合了`recursive_mutex`和`timed_mutex`的功能，当然相对于前三个`mutex`也更加重量级。`recursive_timed_mutex`拥有之前所有`mutex`的所有接口，我们这里不再重复介绍。

## Locks
我们可以通过`mutex`对象的方法来直接操控互斥量，不过需要注意的是在使用完毕后，一定要记得对`mutex`调用`unlock()`方法来释放锁，否则会造成死锁。下面的函数就是一个潜在的造成死锁的例子。

```cpp
std::mutex mtx;

void func(int n)
{
	mtx.lock();
	if (x % 2 != 0)
		throw std::logic_error("not even");

	// 当传入的参数n为奇数时，函数会因为异常而直接退出，下面的unlock没有被执行，造成死锁 
	mtx.unlock();
}
```

我们可以使用`lock_guard`和`unique_lock`对象来管理我们的`mutex`对象，在一定程度上避免死锁的情况。

### std::lock_guard
`lock_guard`的实现很简单，就是在构造函数中对`mutex`对象调用`lock()`，在析构函数中调用`unlock()`。因此，使用`lock_guard`，我们可以保证程序在任意情况下都能释放`mutex`对象，一定程度上避免死锁。

可以将任何继承`BasicLockable`的对象交由`lock_guard`管理。

```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex, std::lock_guard
#include <stdexcept>      // std::logic_error

std::mutex mtx;

void print_even (int x) {
  if (x%2==0) std::cout << x << " is even\n";
  else throw (std::logic_error("not even"));
}

void print_thread_id (int id) {
  try {
    // using a local lock_guard to lock mtx guarantees unlocking on destruction / exception:
    std::lock_guard<std::mutex> lck (mtx);
    print_even(id);
  }
  catch (std::logic_error&) {
    std::cout << "[exception caught]\n";
  }
}

int main ()
{
  std::thread threads[10];
  // spawn 10 threads:
  for (int i=0; i<10; ++i)
    threads[i] = std::thread(print_thread_id,i+1);

  for (auto& th : threads) th.join();

  return 0;
}
```

### std::unique_lock
`unique_lock`和`lock_guard`拥有一样的功能，不同的是`lock_guard`对象仅仅在构造和析构时对`mutex`进行操作，即构造函数中调用`lock()`，析构函数中调用`unlock()`。而`unique_lock`提供了`lock()`和`unlock()`接口，在析构时如果`unique_lock`管理的`mutex`对象已经处于`unlock`状态，那么析构时就什么都不做，否则`unique_lock`会自动释放对应的`mutex`对象。我们可以使用如下方法来构造`unique_lock`对象

```cpp
std::mutex mtx;
std::unique_lock lck1(mtx); // 在构造函数中调用lock()
std::unique_lock lck2(mtx, std::try_to_lock); // 附带一个tag，用来标记当前锁的状态
```

`unique_lock`构造函数的参数除了传入的`mutex`，还增加了一个参数`tag`来初始化当前对象的状态：
- 无`tag`：在构造函数中调用`lock()`方法
- `std::try_to_lock`：在构造函数中调用`try_lock()`方法
- `std::defer_lock`：不调用`lock()`方法，并且认为当前`mutex`处于未上锁状态
- `std::adopt_lock`：不调用`lock()`方法，并且认为当前`mutex`已经被当前线程锁定。

```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread
#include <mutex>          // std::mutex, std::lock, std::unique_lock
                          // std::adopt_lock, std::defer_lock
std::mutex foo,bar;

void task_a () {
  std::lock (foo,bar);         // simultaneous lock (prevents deadlock)
  std::unique_lock<std::mutex> lck1 (foo,std::adopt_lock);
  std::unique_lock<std::mutex> lck2 (bar,std::adopt_lock);
  std::cout << "task a\n";
  // (unlocked automatically on destruction of lck1 and lck2)
}

void task_b () {
  // foo.lock(); bar.lock(); // replaced by:
  std::unique_lock<std::mutex> lck1, lck2;
  lck1 = std::unique_lock<std::mutex>(bar,std::defer_lock);
  lck2 = std::unique_lock<std::mutex>(foo,std::defer_lock);
  std::lock (lck1,lck2);       // simultaneous lock (prevents deadlock)
  std::cout << "task b\n";
  // (unlocked automatically on destruction of lck1 and lck2)
}


int main ()
{
  std::thread th1 (task_a);
  std::thread th2 (task_b);

  th1.join();
  th2.join();

  return 0;
}
```
## 全局函数
### std::lock
`std::lock`用来同时给多个`mutex`上锁，一般情况下，函数会阻塞直到所有的`mutex`都成功调用unlock为止。

如果中途出现任何异常，`std::lock`会将所有已经成功上锁的`mutex`都解锁。所以，使用`std::lock`，结果要么是所有的`mutex`都锁上，要么所有的锁都没有锁上。

函数会根据需要对传入的每一个`mutex对`象调用`lock()`，`try_lock()`，`unlock()`函数，在将所有的`mutex`都锁上之前，避免长时间占用一个`mutex`对象。因此，在两个不同的线程同时调用`std::lock(mut1, mut2)`和`std::lock(mut2, mut1)`不会导致死锁。

```cpp
#include <iostream> // std::cout
#include <thread> // std::thread
#include <mutex> // std::mutex, std::lock


std::mutex foo,bar;


void task_a () {
// foo.lock(); bar.lock(); // 可能造成死锁
std::lock (foo,bar);
std::cout << "task a\n";
foo.unlock();
bar.unlock();
}


void task_b () {
// bar.lock(); foo.lock(); // 可能造成死锁
std::lock (bar,foo);
std::cout << "task b\n";
bar.unlock();
foo.unlock();
}


int main ()
{
std::thread th1 (task_a);
std::thread th2 (task_b);


th1.join();
th2.join();


return 0;
}
```
考察上面被注释掉的代码，如果使用注释中的代码而不是`std::lock`，那么就可能会出现函数`task_a`得到锁`foo`在等待锁`bar`，函数`task_b`得到锁`bar`在等待锁`foo`，这种情况就会出现死锁。

### std::call_once
```cpp
template <class Fn, class... Args>
  void call_once (once_flag& flag, Fn&& fn, Args&&... args);
```
`std::call_once`函数可以用来确保在多线程中，指定的函数只会执行一次。函数一共有3个参数：
- `flag`： `std::once_flag`对象，指定了同一个`flag`的函数只会被调用一次
- `fn`：要调用的函数
- `args`：对应函数需要的参数（可以为多个）

当一个线程A执行到`call_once`函数时，如果该函数从来没有被执行过，并且当前也没有正在运行该函数的线程，那么线程A就会调用`fn`函数。如果当前有线程B正在执行函数`fn`，那么线程A会先进入排队状态而不会执行函数`fn`，直到线程B执行的函数`fn`返回或者触发异常。

当一个线程调用`fn`触发异常后，如果此时还有其他线程在排队执行`call_once`，那么会立即选择下一个线程来执行，直到成功或者所有排队线程都执行过一遍为止。

参考下面的代码：

```cpp
#include <iostream>       // std::cout
#include <thread>         // std::thread, std::this_thread::sleep_for
#include <chrono>         // std::chrono::milliseconds
#include <mutex>          // std::call_once, std::once_flag

int winner;
void set_winner (int x) { winner = x; }
std::once_flag winner_flag;

void wait_1000ms (int id) {
  // count to 1000, waiting 1ms between increments:
  for (int i=0; i<1000; ++i)
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
  // claim to be the winner (only the first such call is executed):
  std::call_once (winner_flag,set_winner,id);
}

int main ()
{
  std::thread threads[10];
  // spawn 10 threads:
  for (int i=0; i<10; ++i)
    threads[i] = std::thread(wait_1000ms,i+1);

  std::cout << "waiting for the first among 10 threads to count 1000 ms...\n";

  for (auto& th : threads) th.join();
  std::cout << "winner thread: " << winner << '\n';

  return 0;
}
```

## 参考资料
> [mutex - C++ Refrence](http://www.cplusplus.com/reference/mutex/)