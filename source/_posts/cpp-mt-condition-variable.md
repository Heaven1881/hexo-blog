---
title: C++11多线程(3)：条件变量
tags:
  - C++11
  - condition-variable
categories: C++11多线程
date: 2018-08-12 10:24:43
---



C++11对多线程进行的支持，我们可以直接跳过系统API直接在语言层面编写多线程程序，直接的好处就是多线程程序的可移植性得到了很大的提高。这篇文章主要介绍**条件变量**相关内容。

<!--more-->

> *正在进行C++11多线程编程相关的学习吗？[点击这里](/categories/C-11%E5%A4%9A%E7%BA%BF%E7%A8%8B/)查看这个分类的更多文章。*

与**互斥量**不同，**条件变量**是用来等待而不是用来上锁的。条**件变量**用来自动阻塞一个线程，直到某特殊情况发生为止。通常**条件变量**和**互斥量**搭配使用。

## 使用场景
我们参考一个生产者和消费者通过队列来传递数据的例子，下面是消费者线程一种可能的实现。
```cpp
std::mutex mtx;
std::list<int> queue;

// 生产者者通过此函数向队列中添加元素
void addToQueue(int n)
{
	std::unique_lock<std::mutex> lck(mtx);
	queue.push_back(n);
}

// 消费者从队列中取出元素
void threadComsumer()
{
	while(true)
	{
		std::unique_lock<std::mutex> lck(mtx);
		while(queue.empty())
		{
			lck.unlock();
			std::this_thread::sleep_for(std::chrono::millseconds(10));
			lck.lock();
		}
	
		int n = queue.front();
		std::cout << n << std::endl;
		queue.pop_front();
	}
}
```
在`threadComsumer()`函数中，消费者通过定期检查队列中是否有数据，如果有则弹出数据，否则就通过`sleep`等待10毫秒，在这里，我们并不知道具体应该`sleep`多长时间。
- 如果`sleep`时间过短，消费者线程就会频繁占用CPU资源，并且频繁占用`mutex`会导致生产者线程不能及时给`mtx`对象上锁。
- 如果`sleep`时间过长，当队列里有元素时消费者就不能及时处理，导致效率上的损失。

那么具体应该`sleep`多长时间呢？我们可以借助条件变量来解决这个问题。

## std::condition_variable
在C++11中，我们可以使用`condition_variable`对象来让当前线程阻塞等待，直到另一个线程**通知**当前线程继续执行。一个`condition_variable`主要有两类方法：

### Wait 类型的方法
Wait类型方法可以让线程进入`sleep`状态，直到满足某些条件：
- `wait`：当前线程进入等待状态，直到另一个线程调用Notify类型的方法
- `wait_for`：当前线程进入等待状态，直到另一个线程调用Notify类型的方法，或者等待时间超过某一个阈值
- `wait_until`：当前线程进入等待状态，直到另一个线程调用Notify类型的方法，或者到达某一个时间点

### Notify 类型的方法
Notify类型方法可以唤醒所有处于`wait`状态的线程：
- `notify_one`：从所有等待的线程中随机选择唤醒一个线程
- `notify_all`：唤醒所有处于等待中的线程
如果当前没有线程处于等待状态，那么notify函数什么也不做。

需要注意的是，如果你使用的条件变量和互斥量是一一对应的，那么使用`notify_one`或者`notify_all`并没有太大的区别。如果你有多个互斥量对应一个条件变量，那么使用`notify_one`并不一定能唤醒正确的线程，导致线程永远等待的问题。这是因为`notify_one`只是随机唤醒一个正在等待对应条件变量的线程，因为这个条件变量对应多个互斥量，所以`notify_one`有概率会唤醒错误的线程，导致异常。而`notify_all`是唤醒所有等待对应条件变量的线程，不会导致

### 使用条件变量
借助条件变量，我们可以将上面的代码改写为：
```cpp
std::mutex mtx;
std::condition_variable cv;
std::list<int> queue;

// 生产者者通过此函数向队列中添加元素
void addToQueue(int n)
{
	std::unique_lock<std::mutex> lck(mtx);
	queue.push_back(n);
	cv.notify_all();
}

// 消费者从队列中取出元素
void threadComsumer()
{
	while(true)
	{
		std::unique_lock<std::mutex> lck(mtx);
		
		while(queue.empty)
			cv.wait(lck);
	
		int n = queue.front();
		std::cout << n << std::endl;
		queue.pop_front();
	}
}
```
使用**条件变量**，我们可以避免固定时间的`sleep`调用，既保证了运行效率，又不会占用额外的CPU时间。

## 参考资料
> [condition_variable -  C++ Reference](http://www.cplusplus.com/reference/condition_variable/)