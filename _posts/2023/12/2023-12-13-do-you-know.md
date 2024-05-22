---
layout: post
title:  "Misc 你知道吗？"
date:   2023-12-13 10:43:01 +0800
tags: misc
typora-root-url: ../../../
---



记录一些想到的问题



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

理论上编译成可执行文件后，global_var的地址不变，但现代操作系统往往有Address Space Layout Randomization ([ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)) 所以实际运行发现会变，```setarch $(uname -m) -R ./a.out``` 手动关闭后地址就不会变。 



## 2. 体系结构

* 当我们讨论CPU cache时，cache的地址是虚拟地址还是物理地址?

如果直接使用PA，则每次需要VA 到 PA 的转换，耗时。如果直接使用VA，则存在不同地址空间相同VA的问题。一般的解决办法(x86)是，Virtually Indexed Physically Tagged (VIPT)。因为cache都比较小，所以可以利用VA和PA中相同的低位去查询cache中内容，同时拿 VA 的VPN 去 TLB 查对应的 PPN，最后比较是否有某个 cache 位置的高位(tag) 和 转换后的地址相同，如果有则hit cache。 VIPT被应用在L1cache，L2/3 cache 使用PIPT。 [参考](https://stackoverflow.com/questions/19039280/physical-or-virtual-addressing-is-used-in-processors-x86-x86-64-for-caching-in-t)

* 现代 x86_64 CPU的cache line是多少？

从物理的角度说，是64 byte，可以写代码验证，不断更新两个在同一个cache line的变量，耗时比变量间隔 64 byte 明显更长(因为 false sharing)。但是一些库(folly/crossbeam-rs)会在代码中设置 128 byte 对齐，而不是 64byte，因为L2 cache 有 prefetcher 可能一次取两个cache line，也会有影响 。

* DRAM是支持随机访问的，那么顺序访问是否和乱序访问速度相同？

不是，DRAM顺序访问是可能比乱序访问快的，因为DRAM中有自己的cache。且DRAM写会比读更慢。此外多线程读DRAM比单线程读DRAM耗时更少，比如读2G数据，两个线程分别读1G的耗时显著比一个线程读2G耗时少，因为平时说DRAM慢主要是latency大，多线程提高了throughput。(todo 之前是看了一篇paper有实验结果的，找不到了)

## 3. 网络

* 现在网站流量那么大，如果每次来数据包都中断一次，这样岂不是切换上下文很多次？

是的，所以Linux有NAPI (new API)机制，处理中断时会poll网卡，询问是否还有数据包要处理，这样可以减少中断次数，提高效率。