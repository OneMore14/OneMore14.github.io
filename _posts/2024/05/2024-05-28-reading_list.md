---
layout: post
title:  "Reading list"
date:   2024-05-28 10:43:01 +0800
tags: list
typora-root-url: ../../../
---



(不代表都仔细看过)



## 0. websites

* [Supertech Research Group](http://supertech.mit.edu/)  The Supertech Research Group investigates the technologies that support scalable high-performance computing, including hardware, software, and theory.
* [REPRODUCING NETWORK RESEARCH](https://reproducingnetworkresearch.wordpress.com/) 研究网络的
* [Johnny's Software Lab](https://johnnysswlab.com/) 关注软件性能
* [uops.info](https://uops.info/index.html) 从x86 microarchitectures 角度分析latency，throughput等性能指标 

## 1. 体系结构

**cache**

[Characterizing Prefetchers using CacheObserver](https://hal.science/hal-03798500/file/sbac-pad22_didier.pdf) 观测Intel L2 cache prefetcher行为



## 2. OS(Linux)

**timer**

[Hashed and hierarchical timing wheels: data structures for the efficient implementation of a timer facility](https://dl.acm.org/doi/10.1145/41457.37504) 讲述分层时间轮算法, tokio的runtime也使用了时间轮。

**调度**

[Earliest Eligible Virtual Deadline First : A Flexible and Accurate Mechanism for Proportional Share Resource Allocation](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564) EEVDF调度器，Linux有参考该算法。和CFS相比，会考虑latency。

**syscall**

[Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf) futex是Linux的一个系统调用，用来帮助实现用户程序的Mutex，早期所有Lock都会进入内核，现在像Rust是先检测是否锁被占用，然后自旋一会，最后进入内核阻塞。



## 3. 分布式

**CAP 理论**

* [Brewer's conjecture and the feasibility of consistent, available, partition-tolerant web services](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf) CAP形式化说明，C要求线型一致性，A表示client请求最后一定能收到有意义回复(报错信息不算)，P表示系统中是否出现分区。CAP中网络模型是异步的，即消息可能延迟任意时长。
* [A Critique of the CAP Theorem](https://www.cl.cam.ac.uk/research/dtg/archived/files/publications/public/mk428/cap-critique.pdf) 批评CAP原文不严谨且与现实系统联系不大，总结consistency和network delay之间的trade-off，且这些成果是早于CAP论文的。 

## 4. Concurrency

**验证并发程序正确性**

* [CDSCHECKER: Checking Concurrent Data Structures Written with C/C++ Atomics](http://demsky.eecs.uci.edu/publications/c11modelcheck.pdf) loom实现参考的paper
* [Partial-Order Methods for the Verication of Concurrent Systems](https://patricegodefroid.github.io/public_psfiles/thesis.pdf) (136页) 减少验证并发的状态空间

## 5. Network

**TCP** 

* [Understanding the Performance of TCP Pacing (2000)](https://homes.cs.washington.edu/~tom/pubs/pacing.pdf) 测量pacing，控制发包速率，避免burst
* [Making Linux TCP Fast](https://netdevconf.org/1.2/papers/bbr-netdev-1.2.new.new.pdf) TCP中关于拥塞控制，发送速率，packet size优化
* [BBR: Congestion-Based Congestion Control](https://queue.acm.org/detail.cfm?id=3022184) 新的拥塞控制算法，计算RTT和bottleneck bandwidth来做拥塞控制
* [TCP Fast Open](https://conferences.sigcomm.org/co-next/2011/papers/1569470463.pdf) 在TCP握手阶段发送数据，降低延迟





