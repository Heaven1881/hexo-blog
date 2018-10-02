---
title: C++11多线程(4)：原子类型
tags:
  - C++11
  - 多线程
categories: C++11多线程
date: 2018-08-25 08:46:35
---


C++11对多线程进行的支持，我们可以直接跳过系统API直接在语言层面编写多线程程序，直接的好处就是多线程程序的可移植性得到了很大的提高。这篇文章主要介绍**原子类型**相关内容。


<!-- more -->

> *正在进行C++11多线程编程相关的学习吗？[点击这里](/categories/C-11%E5%A4%9A%E7%BA%BF%E7%A8%8B/)查看这个分类的更多文章。*

借助**原子类型**，我们可以在多线程编程中用常见的方式访问变量，而不需要额外引入类似与互斥量的保护机制。

原子类型最常见的一个场景就是多线程资源的引用计数问题，一个对象在多个地方被引用，如果涉及到多线程的引用，则需要依赖原子类型来实现引用计数的修改。

## atomic类型
我们可以使用`atomic`模板来将大部分基础类型封装为原子类型。
```cpp
std::atomic<bool> bValue;
std::atomic<int> nValue;
std::atomic<unsigned int> uValue;
```

`atomic`类型同时也提供一系列原子操作，这类操作在CPU中的执行时原子的，我们不需要额外的保护机制。
### is\_lock\_free
获取一个原子类型是否需要依赖互斥锁。该函数的返回值与具体的模板类型和运行平台有关。一个不依赖互斥锁的原子对象不会在使用时导致线程阻塞（类似于`sdt::mutex`对象的`lock`方法）
### store & load
通过`store`设置一个**原子对象**的值，通过`load`取得**原子对象**的值。使用时，我们可以指定内存的同步模式（内存栅栏）。对于`store`，可以使用以下同步模式：
- `memory_order_relaxed`：不进行同步，一般用于引用计数
- `memory_order_release`：仅同步上一次使用`memory_order_consume`和`memory_order_acquire`的`load`原子操作之后的数据操作
- `memory_order_seq_cst`：同步所有数据修改，这是原子操作的默认模式，同时性能消耗也最大

对于`load`可以使用以下同步模式：
- `memory_order_relaxed`：不进行同步，一般用于引用计数
- `memory_order_consume`：仅仅同步对当前**原子类型**变量有依赖的数据操作
- `memory_order_acquire`：同步上一次使用`memory_order_release`和`memory_order_seq_cst`的`stroe`原子操作只有的数据操作
- `memory_order_seq_cst`：同步所有数据修改，这是原子操作的默认模式，同时性能消耗也最大

对于C++11的Memory Order不太清楚的读者，可以参考[这篇文章](http://wilburding.github.io/blog/2013/04/07/c-plus-plus-11-atomic-and-memory-model/)

### exchange
修改**原子对象**的值，然后返回其之前的值。整个操作是原子操作，同事也支持指定内存同步模式，参考`store & load`中关于`memory_order`的介绍。
```
#include <iostream>       // std::cout
#include <atomic>         // std::atomic
#include <thread>         // std::thread
#include <vector>         // std::vector

std::atomic<bool> ready (false);
std::atomic<bool> winner (false);

void count1m (int id) {
  while (!ready) {}                  // wait for the ready signal
  for (int i=0; i<1000000; ++i) {}   // go!, count to 1 million
  if (!winner.exchange(true)) { std::cout << "thread #" << id << " won!\n"; }
};

int main ()
{
  std::vector<std::thread> threads;
  std::cout << "spawning 10 threads that count to 1 million...\n";
  for (int i=1; i<=10; ++i) threads.push_back(std::thread(count1m,i));
  ready = true;
  for (auto& th : threads) th.join();

  return 0;
}
```

### compare\_exchange\_weak
```cpp
bool compare_exchange_weak (T& expected, T val, memory_order sync = memory_order_seq_cst);
```
该方法的语义为：首先将**原子对象**的内容和`expected`变量做比较：
- 如果二者相等，那么将**原子对象**的内容设置为`val`，类似`store`方法
- 如果二者不等，那么将**原子对象**的内容赋值给`expected`，类似`load`方法

上面的操作也是原子的。需要注意的是，函数通过直接比较两个变量的内存数据来判断两个变量是否相等。并不是通过`==`运算符。这类原子操作经常用于无锁队列的实现。

如果**原子对象**和`expected`相等，并且后续的赋值没有失败，则函数返回`true`，否则返回`false`。`compare_exchange_weak`允许在**原子对象**和`expected`参数相等时赋值失败（为了性能考虑），此时返回`false`。

```cpp
#include <iostream>       // std::cout
#include <atomic>         // std::atomic
#include <thread>         // std::thread
#include <vector>         // std::vector

// a simple global linked list:
struct Node { int value; Node* next; };
std::atomic<Node*> list_head (nullptr);

void append (int val) {     // append an element to the list
  Node* oldHead = list_head;
  Node* newNode = new Node {val,oldHead};

  // what follows is equivalent to: list_head = newNode, but in a thread-safe way:
  while (!list_head.compare_exchange_weak(oldHead,newNode))
    newNode->next = oldHead;
}

int main ()
{
  // spawn 10 threads to fill the linked list:
  std::vector<std::thread> threads;
  for (int i=0; i<10; ++i) threads.push_back(std::thread(append,i));
  for (auto& th : threads) th.join();

  // print contents:
  for (Node* it = list_head; it!=nullptr; it=it->next)
    std::cout << ' ' << it->value;
  std::cout << '\n';

  // cleanup:
  Node* it; while (it=list_head) {list_head=it->next; delete it;}

  return 0;
}
```

### compare\_exchange\_strong
`compare_exchange_strong`和`compare_exchange_weak`功能类似，不过前者可以保证在**原子对象**和`expected`参数相等时，函数必定返回`true`，即必定赋值成功。如果实际使用时通过循环赋值，那么还是建议使用`compare_exchange_weak`，后者性能效率更优。

### 数字，指针类原子类型特有操作
这里的操作只允许原子类型为数字或者指针类型，涉及到的操作有：
- `fetch_add`：累加操作，类似于`+=`
- `fetch_sub`：累减操作，类似于`-=`
- `fetch_and`：类似于`&=`，只能用于数字类型
- `fetch_or` ：类似于`|=`，只能用于数字类型
- `fetch_xor`：类似于`^=`，只能用于数字类型

## atomic_flag类型
`atomic_flag`也是一个原子类型，类似于`bool`类型。之所以在这里单独提出来，是因为`atomic_flag`是`C++11`原子类型中唯一一个保证在所有平台都是**无锁的**（不依赖互斥量）。其他`atomic`模板类型并没有无锁的保证，我们可以通过`is_lock_free`来判断一个原子类型是否需要依赖互斥量。

`atomic_flag`类型有两个操作
- `test_and_set`：将对象设为`true`，并返回对象之前的值（`true`或`false`）
- `clear`：将对象设置为`false`

```cpp
#include <iostream>       // std::cout
#include <atomic>         // std::atomic_flag
#include <thread>         // std::thread
#include <vector>         // std::vector
#include <sstream>        // std::stringstream

std::atomic_flag lock_stream = ATOMIC_FLAG_INIT;
std::stringstream stream;

void append_number(int x) {
  while (lock_stream.test_and_set()) {}
  stream << "thread #" << x << '\n';
  lock_stream.clear();
}

int main ()
{
  std::vector<std::thread> threads;
  for (int i=1; i<=10; ++i) threads.push_back(std::thread(append_number,i));
  for (auto& th : threads) th.join();

  std::cout << stream.str();
  return 0;
}
```

## 参考资料
> [atomic - C++ Reference](http://www.cplusplus.com/reference/atomic/)
> [C++11的Atomic和Memory Model的一点认识](http://wilburding.github.io/blog/2013/04/07/c-plus-plus-11-atomic-and-memory-model/)