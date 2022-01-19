---
layout: post
title:  "C++ Cheat Sheet: compile and link "
date:   2021-11-24 18:21:01 +0800
categories: c++
---



# C++ Cheat Sheet: compile & link



本篇主要记录简单的C++编译与链接操作， C++编译器有多种实现，比如g++和clang++，为了少打字，本文使用g++进行实验。



主要是多个源代码文件的编译方式，这里的编译有时候指将C++代码编译为汇编代码，有时候指编译出的二进制文件。



## 1. 编译可执行文件

### 1.1 包含main函数的单文件

众所周知，C++的运行入口是main函数，且有时候也习惯将包含main函数的文件命名为main.cpp。但实际上，包含main函数的cpp文件可以是任何名字。如果main.cpp/a.cpp内容如下

```c++
#include<iostream>
using namespace std;
int main() {
    cout << "hello" << endl;
    return 0;
}
```

通过以下命令将得到可执行文件(默认为a.out)

```shell
g++ main.cpp
```

```shell
g++ a.cpp
```

### 1.2 编译不含main的cpp单文件

#### 编译一个不包含main的cpp文件

如果将上节的main函数改名为f，得到

```c++
#include<iostream>
using namespace std;
int f() {   
    cout << "hello" << endl;
    return 0;
}
```

如果继续编译 ``` g++ f.cpp```，将报错没有main函数

```shell
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/9/../../../x86_64-linux-gnu/Scrt1.o: in function `_start':
(.text+0x24): undefined reference to `main'
collect2: error: ld returned 1 exit status
```

但是，如果有一定基础的话，会知道这里没有出现compile error，并不是编译错误，而实际上是链接错误，g++默认情况下，是想帮我们生成一个可执行文件，但这里的源代码显然是无法直接执行的(没有main函数入口)，因此需要修改编译命令。

我们让g++只编译，不(默认)链接

```shell
g++ -c f.cpp
```

将会得到obj文件 f.o， f.o的内容可以通过objdump工具查看。

#### 链接不包含main函数的obj文件到可执行文件

为了使用编译出来的f.o，需要linker将其链接，首先考虑将其链接到一个可执行文件，在此需要先编译出一个包含main函数的obj文件

```c++
int f(); // 这里的函数声明一定需要
int main() {
   
    f();
    return 0;
}
```

```g++ -c main.cpp```得到main.o，其中main.o调用了包含在f.o里的int f()函数实现，现在需要由linker将它们弄成一个可执行文件

```g++  f.o main.o```或者```g++ main.o f.o``` ，无所谓obj文件的顺序，编译器将把这些obj文件链接到一起，形成可执行文件。甚至也可以obj和cpp文件混用，使用```g++ main.c f.o```或者```g++ main.o f.o```也可以，当然原理其实都一样，只不过g++自己做了这些事情。