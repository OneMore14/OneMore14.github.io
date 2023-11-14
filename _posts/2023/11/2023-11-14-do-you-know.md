---
layout: post
title:  "Misc 你知道吗？"
date:   2023-11-13 10:43:01 +0800
tags: misc
typora-root-url: ../../../
---



## 1. 程序语言

* **C/C++什么时候在全局变量前加 static ? Rust呢?**

参考[这个帖子](https://stackoverflow.com/questions/2271902/static-vs-global)，static 限制了变量仅在本文件内可见。

Rust中可以用来表达 "全局" 的是 ```const``` 或 ```static```，其中每次访问 const 都是自动内联的，详见[文档](https://doc.rust-lang.org/1.30.0/book/first-edition/const-and-static.html)

```rust
struct Node {}

impl Drop for Node {
    fn drop(&mut self) {
        println!("drop")
    }
}

const I: Node = Node {};

fn main() {
    let x = I;
    let mut y = I; // 不会有move的问题，因为这里又新建了一个实例，运行结束有两次drop
    
    let local = Node{};
    let a = local;
    // let b = local; //打开注释会有问题，因为没有 Copy 
}
```

static 则不同，static 变量只会有一个实例，且即使程序结束也不会调用 drop



* **以下代码在编译完成后，反复执行，全局变量的地址是否改变?**

```c++
#include <iostream>

int global_var = 10;
int main() {
    printf("addr of global_var %p\n", &global_var);
}
```

理论上编译成可执行文件后，global_var的地址不变，但现代操作系统往往有Address Space Layout Randomization ([ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)) 所以实际运行发现会变，```setarch $(uname -m)  ./a.out``` 手动关闭后地址就不会变。 
