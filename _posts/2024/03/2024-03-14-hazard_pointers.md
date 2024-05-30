---
layout: post
title:  "Hazard Pointers"
date:   2024-03-14 10:43:01 +0800
tags: concurrency
typora-root-url: ../../../
---


了解hazard pointer，在阅读 *Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects* 之前，先阅读其用来举例的文章 *Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms*，之后可以做 https://github.com/kaist-cp/cs431 ，其中有一个编程作业就是简单版的hazard pointer，要求代码放在私有仓库，所以我的代码没有公开。

# Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms

## 2. Algorithms

数据结构定义如下

![image-20240205180824497](/assets/2024/03/hp1.png)

指针除了指向节点外，还有一个count记录修改次数用来避免ABA问题。

![image-20240205181419572](/assets/2024/03/hp2.png)

初始化，head和tail都指向dummy节点。

![image-20240205181515204](/assets/2024/03/hp3.png)

数据入队，首先创建一个节点，next为NULL，获取当前tail，并记当前tail的next指针为next变量。如果tail变量没变，仍然是Q的tail，分两种情况: 如果tail指向最后一个节点，那么尝试将node加在最末尾，这样tail指向倒数第二个节点，结束。否则tail不是指向最后一个元素，将tail通过CAS指向下一个元素。 循环直到node被加在队列尾，然后将tail指向新的node。

![image-20240205182253969](/assets/2024/03/hp4.png)

pvalue只是为了获取dequeue出来的值，首先和enqueue类似，先校正tail的位置，然后用CAS修改head。

## 3. Correctness

### 3.1 Safety

上面算法是安全的，因为满足以下特性

1. The linked list is always connected.
2. Nodes are only inserted after the last node in the linked list.
3. Nodes are only deleted from the beginning of the linked list.
4. Head always points to the first node in the linked list.
5. Tail always points to a node in the linked list.(没有说指向最后一个节点)

在初始化时，这些性质全部成立。接下来在不考虑ABA问题的情况下，证明这些性质一直成立。

1. The linked list is always connected because once a node is inserted, its next pointer is not set to NULL before it is freed, and no node is freed until it is deleted from the beginning of the list (property 3). (有点迷，因为新节点的next.ptr是NULL)
2. 节点只会被插入到链表最后，因为是通过tail指针连接的，并且只会连在一个next为null的节点上。
3. 节点只会从第一个元素被删除，因为永远是根据head来删的，而head指向第一个元素。
4. head永远指向第一个元素，不会为null，因为有dummy节点
5. tail永远指向某个元素，因为tail一直在head后面，所以tail不会指向被删除的元素，且当next为null的时候，tail的值不会改变。

### 3.2 Linearizability

The presented algorithms are linearizable because there is a specific point during each operation at which it is considered to “take effect”.  比如入队是在将新节点插入队列时，出队是移动head时。

### 3.3 Liveness

略，也只是case by case分析一直循环的原因是什么，没有严格形式化证明。



这个lock free不能应用在实际生产中，因为dequeue在free(head) 时，可能有其他线程读到了原来的head，然后停在D3，释放后其他线程会继续访问这个head。详见 https://stackoverflow.com/questions/40818465/explain-michael-scott-lock-free-queue-alorigthm



# Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects


## 1. INTRODUCTION

提出一种wait-free的方法，可以防止出现lock-free中关于内存的问题(比如上面的问题)。同时可以保证将不使用的内存返还给OS。

A hazard pointer either has a null value or points to a node that may be accessed later by that thread without further validation that the reference to the node is still valid. Each hazard pointer can be written only by its owner thread, but can be read by other threads. 也就是说，hazard pointer中存放了我们还需要访问的内存地址，当我们要free一个节点时，如果还有hazard pointer指向它，就不能free。

当一个线程要free节点时，先将其加入线程私有的队列rlist(retire list)，暂不直接free，而是等队列大小达到一定容量后，扫描所有hazard pointer，如果待释放节点没有hazard pointer指向它，就可以真正释放，否则就还是留在rlist。

## 2. PRELIMINARIES

如果GC可以阻止ABA问题，那么hazard pointer也可以。

## 3. THE METHODOLOGY

### 3.1 The Algorithm(basic)

![image-20240206160534690](/assets/2024/03/hp5.png)

每个线程一个HPRecType，其中K个*NodeType就是最多可以容纳K个hazard pointer。

![image-20240206161620839](/assets/2024/03/hp6.png)

每当要释放一个节点的时候，调用retireNode()，将其加入rlist.

![image-20240206161801872](/assets/2024/03/hp7.png)

scan()做的事情其实很简单，遍历所有hazard pointer，然后再匹配自己的rlist的，如果有hazard pointer指向rlist中的节点，那么这个节点不会被释放，而是继续留在rlist。

### 3.2 Algorithm Extensions



![image-20240206162935957](/assets/2024/03/hp8.png)

算法扩展可以不用提前知道线程的数量，由于线程的加入和离开是动态的，所以可以考虑重复使用HPRecType，其中增加一个active变量，代码当前实例是否在被使用。当线程离开时，它的rlist可能还不为空，因此将rlist移入HPRecType，这样线程离开时可以将其转交给其他线程。 

一个节点有以下状态

1. *Allocated:* n is allocated by a participating thread, but not yet inserted in an associated object.
2. *Reachable:* n is reachable by following valid pointers starting from the roots of an associated object.
3. *Removed:* n is no longer reachable, but may still be in use by the removing thread.
4. *Retired:* n is already removed and consumed by the removing thread, but not yet free.
5. *Free:* n’s memory is available for allocation.
6. *Unavailable:* all or part of n’s memory is used by an unrelated object.
7. *Undefined:* n’s range of memory locations is not currently viewed as a node.

***Own:*** A thread $$j$$ owns a node $$n$$ at time $$t$$, iff at $$t$$, $$n$$ is allocated, removed, or retired by j. Each node can have at most one $$owner$$.

***Safe:*** A node $$n$$ is safe for a thread $$j$$ at time $$t$$, iff at time $$t$$, either $$n$$ is reachable, or $$j$$ owns $$n$$.

***Possibly unsafe:*** A node is possibly unsafe at time $$t$$ from the point of view of thread $$j$$, if it is impossible solely by examining $$j’s$$ private variables and the semantics of the algorithm to determine definitely in the affirmative that at time $$t$$ the node is safe for $$j$$.

***Access hazard:*** A step $$s$$ in thread $$j’s$$ algorithm is an access hazard iff it may result in access to a node that is possibly unsafe for $$j$$ at the time of its execution.

todo 后面的分析太复杂了

