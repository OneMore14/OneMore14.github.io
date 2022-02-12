---
layout: post
title:  "Paxos Made Simple"
date:   2022-02-11 10:34:01 +0800
categories: "6.824 分布式"
---

The Paxos algorithm, when presented in plain English, is very simple.   --Leslie Lamport

## 1 Introduction

## 2 The Consensus Algorithm
### 2.1 The Problem
假设有一系列进程能propose values，一致性算法需要保证只有一个值被选中；如果没有值被提出，就没有值被选中；如果一个值已经被选中，那么这些进程应该能知道被选中的值。

一致性的安全要求是:

* Only a value that has been proposed may be chosen
* Only a single value is chosen, and
* A process never learns that a value has been chosen unless it actually
has been.

目标是保证某个被提出的值最终被选中，且进程能最终知道这个值。

一致性算法中有3种角色:
* proposers
* acceptors
* learners

在实现中，一个进程可能会扮演多个角色。假设角色间可以互相通信，我们使用典型的异步、非拜占庭模型:

* 所有角色按任意速度运行，可能停止或重启。由于所有角色可能都在某个值被选中后重启，因此，某个角色必须在宕机重启的过程中记住某些信息(持久化)，否则无解 

* 消息可以等任意时间后送达，可以重复或丢失，但不会被纂改

### 2.2 Choosing a Value

*(注: 论文中有两个概念需要明确, Accept(接收)指Acceptor确认收到了某个提案；Chosen(选中)指某个提案被整个系统确认为最终的一致性提案。被Accept的提案不一定被Chosen，而被Chosen的提案一定是被Accept过的)*

最简单的办法是只有一个acceptor，proposer向acceptor发送提案，acceptor选择接收最先收到的值。这种办法非常简单，但acceptor宕机后整个系统就不可用了

所以需要设置多个acceptor。一个proposer向多个acceptor发出提案的值。一个acceptor可能会接受这个值。只有当足够多的acceptor都接受了这个值，才能认为系统最终选中了这个值。为了保证只有一个值最终被选中，上文中的足够多实际要求是大于一半。

在没有宕机或消息丢失的情况下，即使只有一个proposer我们也希望能最终选中一个值，这要求以下条件

* P1. An acceptor must accept the first proposal that it receives.

但这个条件引入新的问题，几个值可能几乎同时被提出，每个acceptor接受了不同的值，导致没有大多数产生。哪怕只有2个提案，当一个acceptor宕机后，也可能每个提案恰好被一半的acceptor接受，没有产生大多数。

P1和每个值必须被大多数acceptor接受的条件暗示每个acceptor必须接受不止一个提案。我们让acceptor给接受的多个提案分配一个数字(natural number)，这样每个提案实际包含proposal number和value两个部分。为了防止迷惑，要求不同提案需要有不同的编号，编号如何产生依赖于实现，当前只需要假设有这样的编号机制。

*(注: 为什么上文说暗示每个acceptor必须接收不止一个提案? 假设一共有3台acceptor，其中一台宕机，还剩下两台运行，记为acceptorX和acceptorY，此时还没有任何提案发出。假设有一个proposer此时提出提案A发给X和Y，但发给Y的数据丢失，只有X收到A。此后另一个proposer发出提案B，同时发给X和Y且均正常到达。如果acceptor只能接收一个提案，那么结果是X收到A，Y收到B，没有提案能被选中。如果acceptor可以接收多个提案，X就能接收A和B，Y能接收B，最终选中提案B)*

*(注: 编号的实际实现方式可以是 时间戳+proposerId 的组合，这样不同proposer的提案id一定不同，且按时间顺序单调递增)*

可以允许多个提案被选中，但前提是这些提案的值都一样。再结合提案编号，有如下条件需要满足

* P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.

*(注: 为什么更高编号的提案值只能和已经被选中的值保持一致?因为paxos是目的就是让多个进程对某一个单值达成一致，不同提案中的值并没有优劣或大小或新旧之分，后发起的提案并没有"更新"这个说法，既然已经有提案被选中，就完全没有必要再去修改被选中的值，因此只需要保持一致即可)*

因为编号是完全有序的，P2就保证了只有一个value被选中的安全性

一个提案至少要被一个acceptor接受才可能被最终选中。因此，我们可以通过满足P2(a)来满足P2

* P2(a) If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.

此时仍然保留P1的约束，用于保证一定有某个提案被选中。由于通信是异步的，一个提案可能被某个没有接收到任何提案的acceptor(记为c)所接收。假设有一个新的proposer“醒来”，发起一个更大编号且不同值的提案，那么条件P1要求c必须接受这个提案，违反了条件P2(a)。想要同时保证P1和P2(a)需要把P2(a)加强为P2(b)

* P2(b) If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v.

由此可以从P2(b)推导出P2(a)，P2(a)再推导出P2。

我们考虑如何满足P2(b)，首先考虑证明如何保持P2(b)。假设有一个编号为m值为v的提案被选中了，我们将证明一种proposer发起提案的算法，使n > m的提案也具有值v。我们使用数学归纳法，即假设m..(n - 1)的提案值也全部是v。当提案m被提出且选中时，一定存在一个集合C，这个集合由大多数Acceptor组成，且C中的Acceptor全部都接收了提案m。那么C中所有Acceptor都会接收m到n-1的提案，且这些提案的值都是v。那么这时任意一个包含大多数Acceptor的集合S，其中某个Acceptor一定也属于集合C。通过维护P2(c)的性质，我们可以归纳出提案n的值是v

* P2(c) For any v and n, if a proposal with value v and number n is issued,
then there is a set S consisting of a majority of acceptors such that
either (a) no acceptor in S has accepted any proposal numbered less
than n, or (b) v is the value of the highest-numbered proposal among
all proposals numbered less than n accepted by the acceptors in S.

*(注: P2(c)是怎么推导出P2(b)的？当提案m被选中后，任意S集合一定有C集合中的acceptor，而这个acceptor接收的提案m就是编号最大的提案)*

为了维护P2(c)的性质，当一个proposer想要发起提案n时，必须知道小于n的最大编号提案是否已经或将要被大多数acceptor所接收。查询是否已经被接收很容易，而预测很难。那么干脆不预测，而是控制其一定不会被接收。也就是说，proposer要求acceptor不要再接受编号小于n的提案，于是有如下proposer的算法:

1. A proposer chooses a new proposal number n and sends a request to
each member of some set of acceptors, asking it to respond with:

    (a) A promise never again to accept a proposal numbered less than n, and
    
    (b) The proposal with the highest number less than n that it has accepted, if any.

这样的请求被称为prepare request with number n

2. If the proposer receives the requested responses from a majority of
the acceptors, then it can issue a proposal with number n and value
v, where v is the value of the highest-numbered proposal among the
responses, or is any value selected by the proposer if the responders
reported no proposals.

第2阶段请求称为accept request

目前描述了proposer的算法，那么acceptor呢？acceptor可以接收prepare request和accept request。acceptor可以忽略任何违反了一致性安全的请求。acceptor总是可以回复prepare request，但只有在没有违反承诺的情况下接受accept request。也就是可以说成P1(a)

* P1(a) An acceptor can accept a proposal numbered n iff it has not responded
to a prepare request having a number greater than n.

P1(a)可以保证P1

目前已经有了完整的算法，最后引入一些小优化。acceptor如果已经回复了一个大于n的prepare request，之后再收到n的prepare request，就没有必要再处理后来的这个n prepare request。同样，如果已经接收了提案n, 也可以忽略id为n的prepare request。

有了这种优化后，acceptor只需要记住已经接收的最大提案编号和已经回复的最大prepare request编号。因为P2(c)需要即使宕机也能保证性质，这些信息需要持久化。而proposer却不需要记住这些信息。

结合proposer和acceptor，我们可以得到如下两阶段算法

* Phase 1. (a) A proposer selects a proposal number n and sends a prepare
request with number n to a majority of acceptors.

    (b) If an acceptor receives a prepare request with number n greater
    than that of any prepare request to which it has already responded,
    then it responds to the request with a promise not to accept any more
    proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted.

* Phase 2. (a) If the proposer receives a response to its prepare requests
(numbered n) from a majority of acceptors, then it sends an accept
request to each of those acceptors for a proposal numbered n with a
value v, where v is the value of the highest-numbered proposal among
the responses, or is any value if the responses reported no proposals.

    (b) If an acceptor receives an accept request for a proposal numbered
    n, it accepts the proposal unless it has already responded to a prepare
    request having a number greater than n.

一个proposer可以提出多个提案，只要每一个提案遵守算法规则即可。而且proposer也可以在任意时间丢弃提案。acceptor在收到为n的prepare request时候，如果已经提前收到大于n的prepare request，可以通知proposer丢弃这个提案，这算是一个不影响正确性的性能提升。

### 2.3 Learning a Chosen Value
为了知道哪个值最终被选中了，learner必须找出一个被大多数acceptor所接受的提案。最显然的算法是每次acceptor接收一个提案后，将这个消息发给每一个learner。这样做却需要大量的通信，一个简单的做法是在learner中选出一个leader，acceptor只发消息给leader，由leader再分发消息。但这样leader宕机后整个系统不可用，进一步优化则是同时维护一组leader。*(注: 这里体现出系统设计中的trade off)*

### 2.4 Progress

很容易构造出一种没有进展的场景，假设有两个proposer分别为p和q，p先发起prepare request 1，在p发起accept request 1之前，q又发起了prepare request 2，这时p的accept request 1不会被接收，之后在q的accept request 2发起前p又重新发起prepare request 3...这样下去永远不会有进展。

为了保证有进度，必须在proposer中选出一个leader，只有leader能发起提案。通信正常情况下，总能最终选中一个值。

### 2.5 The Implementation

Paxos算法中，每个进程都扮演proposer，acceptor和learner。算法选举一个proposer leader和learner leader。Paxos中，request和response都作为普通消息传递。需要有稳定存储保留acceptor需要记住的信息。Acceptor在发出回复前先将回复记录在存储介质中。

为了保证两个提案的编号一定不同，不同的proposer从不同的集合中选择数字作为编号，且每个proposer需要持久化它所发起的最大提案编号。

## 3 Implementing a State Machine

实现分布式系统的一种简单方式是将其作为一系列客户端向中心服务器发送命令。服务器可以被看作是确定状态机，按某种顺序执行客户端命令。使用单个中心服务器有单点故障的风险，因此准备一套服务器，每个服务器只要按相同顺序执行命令，就总会产生同样的结果 *(注: 因为是确定状态机)* 

为了让所有状态机按同样顺序执行命令，我们实现一整套独立的Paxos实例，第i个Paxos实例选中的值就是第i个将要执行的命令。现在，假设服务器是固定的，所有Paxos实例使用同一套agents

在普通情况下，一台server被选举为所有Paxos实例的leader。客户端向leader发送命令，由leader决定命令的顺序。如果leader决定一条命令应当在135th的位置被执行，那么leader会尝试在第135个实例中选中该条命令。这种尝试大部分情况会成功，但也可能因为宕机或其他server任务自己是leader而失败，但一致性算法保证了最多只有1条命令在第135的位置被执行。

这种办法的关键在于，在Paxos一致性算法中，将要提出的提案值只有到了phase2才能确定。回顾一下，在phase1结束后，这个值要么已经决定了，要么可以是任何值。

接下来描述leader宕机后，新leader被选举后的场景。

新leader之前是learner，应当知道大多数已经被选中的值。假设新leader已经知道1-134,138和139的选中命令(下文将描述为何有间隔)。那么它接下来会为135-137和大于139的实例执行phase1。假设执行后只得到了135和140的结果，但不能确定136和137应当是哪条命令 *(注: 不是很理解为什么136和137不能获得结果，猜测是136和137的Phase1回复中，acceptor还没有接收任何提案，所以新leader不知道应该是什么命令)*

目前为止，新leader和其他server一样，知道了1-135的命令，却不能执行138-140的命令，因为136和137是未知的。新leader可以把客户端接下来的两个命令填充到136和137，但实际上我们让136和137填充一个特殊的命令"no-op"，这样使状态不变，然后再执行138-140，再之后，新leader就可以正常从client收到命令后，按自己想法填充之后的所有命令。

然后解释为什么会有间隔存在，因为leader可以在第n个命名确定被选中之前就发起第n + 1轮提案。leader所有关乎第n轮提案的消息可能全部丢失，而这时第n + 1轮已经确定，正常情况下leader会为第n轮提案重发消息，但如果这时leader宕机，就没有server知道第n轮提案中到底是什么命令。

当一个新leader被选举后，它可以执行任意轮phase 1。在上面的场景中，执行了135-127和所有大于139的轮次。使用相同的提案id，新leader可以向其他server发送一条简短的消息，这时的acceptor如果在之前已经收到过某个proposer的phase 2消息，那么acceptor不会仅仅回复一个OK，而是带上之前的提案，比如上文中的135和140.

目前只讨论了一切正常，只有一个leader存在的情况(除了leader切换过程中有一小段时间没有leader)。在不正常情况下，有可能选举失败，或者有多个server认为自己是leader，那么这两种情况都有可能让系统不能选中值而停止进度，但并不会破坏一致性。单一leader只是保证了make progress

