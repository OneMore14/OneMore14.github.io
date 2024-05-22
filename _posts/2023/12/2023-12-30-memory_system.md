---
layout: post
title:  "Memory System"
date:   2023-12-30 10:43:01 +0800
tags: architecture
typora-root-url: ../../../
---



### 1. 对程序性能的疑惑

程序员对 cache 的结构都比较熟悉，但是介绍内存实现的文章不多，突然想探索内存实现是遇到下面这个问题:


假设有一个大数组，如果用一个线程读取每个数组元素，示例代码如下

```c++
for (int i = 0; i < N; i++) {
    int value = arr[i]
}
```

再假设上面这段代码耗时10秒，如果我们将数组一分为二，开两个线程去读

```c++
for (int i = 0; i < N / 2; i++) { // 线程1 读数组前一半
    int value = arr[i]
}

for (int i = N / 2; i < N; i++) { // 线程2 读数组后一半
    int value = arr[i]
}
```

我们会发现，开两个线程读完整个数组的耗时会减少一半，即耗时5秒左右。

开两个线程让时间减半好像很合理，但又好像不太对......我们常说，CPU比内存要快的多，那为什么单线程的时候没有“打爆”内存呢，反而开两个线程得到成倍的性能提升，难道CPU比内存慢？

### 2. 内存的结构

首先，我们将讨论范围限定在[SDRAM(Synchronous dynamic random-access memory)](https://en.wikipedia.org/wiki/Synchronous_dynamic_random-access_memory) ，就是现在计算机中常见的“内存条”。dynamic指RAM需要动态刷新，因为DRAM用一个电容的状态表达一个bit的值，电容会慢慢漏电，需要在这个bit的值消失前重新上电，刷新间隔时间通常以毫秒计，在刷新时内存无法处理任何读数据或写数据的请求。synchronize指内存的操作接口按外部时钟同步。另外现在的内存大多满足 DDR(Double Data Rate) 标准，DDR指信号可以在时钟的上升沿和下降沿各发送一次，这样在内存频率不变的情况下，相同时间内传输的数据可以增加一倍。厂商们在产品上宣称DDR 3200 有 3200MHz的频率，实际上只是1600MHz × 2。DDR的data bus有64个位，即一次读取64个bit，假设缓存向内存读取一个64 byte大小的 cache line，那么使用DDR的话，需要传输4个周期的数据(因为一个周期读 2 × 64 bit)。



一块内存通过主板和一个memory controller相连，memory controller向内存发送读写指令，现代处理器可以有多个memory controller，也就可以插多个内存，这种结构称为多通道(channel)。

我们常说的一块“内存条”，实际术语叫**Dual Inline Memory Modules(DIMMs)**，由多个 **rank** 组成，rank数一般是1，2或者4。每个rank相对独立。一个 rank 由多个 DRAM chip组成，假设一个DRAM chip 的数据宽度为 8bit，写作x8 chip，由于DRAM本身要求一次传输64 bit数据，就需要一个rank由 64 ÷ 8 = 8个 chip组成。类似的，如果是x2 chip, 一个rank就需要32个chip。 这里也可以看出，取64 bit是从1个rank里取的。如果再补充从rank里取数据会花很长时间，自然而然地，我们会想DRAM有没有可能一次从多个rank中取数据来提高性能？答案是肯定的，前一个读取请求还在rank1中处理时，如果有另一个读rank2的请求，则不会等第一个请求结束才开始读rank2，而是并行执行，非常类似CPU中的多核处理。

然而rank还不是DRAM中的最小单位，rank 被划分为多个 bank，每个bank可以独立处理请求，每次读一个cache line(64 bytes)实际上也是只从一个bank读。假设一个rank是8个×8 chip组成的，那么bank也被划分为8个部分，每个部分各存在于其中一个chip上。每次传输的64 bit，实际上分散在这8个 x8 chip中，而不是从一个chip读的。

每个bank 再被划分为多个 array，每个array形如一个二维数组。内存读取数据也会先根据行地址选出一行，再选择这一行的某几列数据。假设一个array是由1024行和1024列组成，则一共有1M bit的数据。内存每次读数据时，会将选出来的一行数据全部作为缓存，假设cache line需要64个array，则一共缓存64 * 1024 bit = 8k bytes。注意，一个array只缓存一行，因为array中的数据不是连续的，不能想象成编程语言的数组，前面也说了读取的64bit数据分散在多个DRAM chip上。

为了利用访问内存的locality，可以将地址按下面这样翻译:

row : rank : bank : channel : column : block offset



回到原来的问题，两个线程能提速是利用了DRAM的并行处理能力。

