---
layout: post
title:  "阅读 C++ Concurrency in Action"
date:   2023-11-23 10:43:01 +0800
tags: cpp
typora-root-url: ../../../
---

不深究C++语法细节，更多关注与Rust的共性部分。



## Ch2 Managing threads

### 2.1 Basic thread management

#### 2.1.1 Launching a thread

如果不等待线程结束的话，需要保证线程访问的数据一直有效，这是C++中容易出现的问题(比如用到了某个指向本地变量的指针)，Rust在spwan的时候限制了trait bound ```'static``` 和 ```Send```

```c++
void f(int* x) {
    while (true) {
        *x += 1;
        std::cout << *x << std::endl;  // 指针会在中途失效
        sleep(1);
    }
}

void g() {
    int x = 0;
    std::thread t = std::thread(f, &x);
    t.detach();
}
```

#### 2.1.2 Waiting for a thread to complete

join当然可以等待线程结束，但是需要更细致的控制可以使用condition variables 或 futures

#### 2.1.3 Waiting in exceptional circumstances

如果要detach线程，最好在子线程开始后立即detach。如果要join等待，要考虑执行到join前会不会发生异常，避免子线程访问已经不存在的数据，解决办法是使用RAII将thread再包一层，在析构函数中调用join。C++ 中要注意 join 只能被调用一次，Rust不需要考虑这个问题，因为join函数是```join(self)``` 

#### 2.1.4 Running threads in the background

### 2.2 Passing arguments to a thread function

除了变量中途不可访问外，还有可能相反，函数要求一个引用，却隐式地传进了一个复制的新变量。

```c++
void update_data_for_widget(widget_id w,widget_data& data);
void oops_again(widget_id w)
{
	widget_data data;
	std::thread t(update_data_for_widget,w,data); // copy了一个data，而不是原来的
	display_status();
	t.join();
	process_widget_data(data);
}
```

rust没有这个问题，因为```T``` 和 ```&T``` 分的很清楚，语法上不能把```data```作为引用传参，必须用```&data```， 类似的，C++用```std::ref(data)```

### 2.3 Transferring ownership of a thread

C++也有move，但Rust有```Send```和```Sync``` ，对多线程的限制更严格，比如强制Rc不能跨线程。

### 2.4 Choosing the number of threads at runtime

C++有```std::thread::hardware_concurrency()``` Rust有对应的 ```std::thread::available_parallelism()``` ，没有设置CPU亲和性的场景下，一般是CPU核数。

### 2.5 Identifying threads

Rust可以在用Builder的时候指定线程名称。



## Ch3 Sharing data between threads
### 3.1 Problems with sharing data between threads

#### 3.1.2 Avoiding problematic race conditions

1. 添加保护机制，保证一次只有一个线程做操作。
2. 设计好的数据结构，让其自己每次操作是独立的(lock-free programming)
3. software transactional memory(STM) 将每次数据修改当做事务

### 3.2 Protecting shared data with mutexes

C++需要显示使用lock_guard，Rust的mutex就只支持类似的RAII用法。

假设有一个类，简单粗暴地给类函数每次访问成员变量时都加锁并不一定总是OK，如果某个函数返回值是类成员变量的指针，那一切的加锁都没有意义了。此外，也有可能将成员变量隐式地传递出去，比如某个函数将成员变量指针保存在全局变量中。

*Don’t pass pointers and references to protected data outside the scope of the lock, whether by returning them from a function, storing them in externally visible memory, or passing them as arguments to user-supplied functions.* 同样适用于Rust



即使每一个操作都是线程安全的，但把它们组合起来也未必是安全的。比如

```c++
stack<int> s; // 即使 empty() top() pop() 都是线程安全的，但组合在一起就没有保证了
if(!s.empty()) {
	int const value=s.top();
	s.pop();
	do_something(value);
}
// 那么考虑将top() 和 pop() 合并在一个接口呢？也不行，因为copy constructor可能会抛异常
```



## ch5. The C++ memory model and operations on atomic types

rust 的memory model基本也是抄的C++，这里讲述c++11的内存模型

