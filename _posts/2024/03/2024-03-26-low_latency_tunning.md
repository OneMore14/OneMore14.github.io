---
layout: post
title:  "low latency tunning"
date:   2024-03-26 10:43:01 +0800
tags: performance
typora-root-url: ../../../
---



一些资料

* [Low Latency Performance Tuning for Red Hat Enterprise](https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf)
* [Low latency tuning guide](https://rigtorp.se/low-latency-guide/)
* [Optimizing Computer Applications for Latency: Part 1: Configuring the Hardware](https://www.intel.com/content/www/us/en/developer/articles/technical/optimizing-computer-applications-for-latency-part-1-configuring-the-hardware.html)
* [Optimizing web servers for high throughput and low latency](https://dropbox.tech/infrastructure/optimizing-web-servers-for-high-throughput-and-low-latency)



## 1. Hardware tuning

### 1.1 Disable hyper-threading

减少对CPU竞争，也减少对cache的竞争

### 1.2 Enable Turbo Boost

开启超频，但是需要好的冷却设备

## 2. Kernel tuning

### 2.1 Use the performance CPU frequency scaling governor

通过内核设置CPU性能模式。

### 2.2 Isolate cores

绑定核, 关闭swap

## 3. programming tricks

### 3.1 branch

考虑使用 **likely** / **unlikely**，在编译器层面上优化，intel已经没有static branch hints。

在stable rust中可以使用#[cold]，标记在函数上，提示llvm这个函数很少被调用，llvm可能因此做一些编译上的调整。

**const generics**

对于rust，可以考虑将函数入参移到const generics位置，可以减少分支，详见 https://blog.cocl2.com/posts/const-currying-rs/





还包括大页(减少TLB cache miss)，合理使用cache，避免false sharing，减少锁竞争，合理使用SIMD，网络参数设置等等，还是要看实际情况。





