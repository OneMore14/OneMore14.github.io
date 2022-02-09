---
layout: post
title:  "Raft论文 In Search of an Understandable Consensus Algorithm"
date:   2021-12-02 10:34:01 +0800
categories: "6.824"
---


### 1. Introduction

Paxos很牛，统治了一致性算法领域，但学生非常难以理解，工程师也非常难以实现。本文的目的是为了让Paxos好懂。

我们的算法就是Raft

### 2. Replicated state machines

![image-20211202103939301](/assets/2021/12/raft/image-20211202103939301.png)

一致性算法主要存在于replicated state machine场景中。如图一，每个server有自己的command log，server接收到请求后将命令按顺序写入log，然后状态机按log执行命令，如果每个server的command log是一样的，那么状态机的状态也应当是一样的。一致性算法的作用就是使每台server的command log保持一致，一致性模块需要从client接收请求并将请求写入command log，同时需要和其他server的一致性模块通信，保证每台server的command log有相同的命令，且命令的顺序也相同，即使其中一些server已经宕机。保证了command log的一致性，自然也保证了对client输出的一致性

现实系统的一致性算法通常有以下几个特点

* 在non-Byzantine条件(包括网络延迟、分区、丢包、包重复、包乱序)下，确保safety(绝不返回错误的结果)
* 只要大多数服务器是正常的，集群就能提供完全可靠的功能，2n + 1个server最多可容忍n个宕机，宕机的服务器可以恢复后再加入集群
* 不利用timing保证日志的一致性，因为错误的时钟和消息延迟可能会导致可用性问题
* 通常情况下，集群中多数server响应后，命令被认为已经完成，少数的慢server不会影响系统整体的性能

### 3. What’s wrong with Paxos?

Paxos太难理解，太难实现，于是作者提出了Raft。

### 4. Designing for understandability

除了保证算法本身的完整性和可用性，本文特别注重让算法容易理解，为此将很多部分拆分来讲解。

### 5. The Raft consensus algorithm

Raft是一种管理上文中提到了replicated log的算法，图二是对算法的总结，图三列出了算法的主要特点



![image-20211202140227063](/assets/2021/12/raft/image-20211202140227063.png)



![image-20211202140521717](/assets/2021/12/raft/image-20211202140521717.png)

Raft实现一致性的方法是首先为replicated log选举一个leader，然后给leader所有的权限去管理replicated log。leader从客户端接受log，复制这些log到其他server，然后告诉其他server在何时可以将这些log应用到各个server的状态机。有一个leader可以简化对状态的管理，反正都是leader说了算，而且如果leader宕机或失去连接，还可以再重新选举一个新leader。有了leader这种机制后，raft将一致性问题解耦成几个相对独立的问题:

* Leader election
* Log replication leader必须从client接收log，然后强制让集群中其他server也接受该log
* Safety Raft关键的安全特性就是图三中状态机的安全特性。

#### 5.1 Raft basics

Raft集群有多个服务器，一般是5。在任何时候，一个server处于以下三种状态之一

* leader
* follower
* candidate

一般地，只有一个leader存在，其他的都是follower。Follower是被动的，它们不会提出request，而是不断回应leader和candidate的询问。leader会处理client请求，如果client联系了一个follower，那么follower将连接重定向到leader。Candidate角色是用于选举一个新leader。图四显示了它们之间的转换关系。

![image-20211202142457325](/assets/2021/12/raft/image-20211202142457325.png)



Raft把时间分成任意长度，每一段称为term。terms被标记上连续的数字，每个term从选举开始，如果有一个candidate赢得了选举，那么在当前term之后的剩余时间，它都是leader。如果一场选举导致split vote，这种情况下term以没有leader结束，一个新term会开始，Raft保证在一个term内最多只有一个leader(可能没有)

![image-20211202142629018](/assets/2021/12/raft/image-20211202142629018.png)

在不同的时间，不同的server可能会看到在不同的term之间转移，并且在某些时候，一个server可能观察不到选举事件甚至观察不到整个term的发生。在Raft中，term像是逻辑时钟，每个server记录自己当前的term编号，term编号会周期性地增加。一个server的当前term编号信息会和其他server交换，如果一个server的term编号比其他server的小，那么它将更新到更大的term编号。如果一个candidate或leader发现自己的term编号过期了，那么它们会立即更换成follower身份。如果server接受到一个term编号更小的请求，则直接忽略请求。

Raft协议中各server通过RPC通信。

#### 5.2 Leader election

​        Raft使用心跳机制触发leader选举。当servers启动时，它们是followers。一个server如果持续从leader或candidate收到RPC请求，则其保持follower身份。Leader会周期性地给所有follower发送心跳RPC，以宣示自己的存在。如果一个follower在一段时间(election timeout)没有收到请求，就会认为当前没有leader，于是开始选举程序。

​        开始选举后，follower增加自己当前的term编号，并且进入candidate状态。然后它为自己投票，并向集群其他服务器发送 RequestVote RPC，一个candidate会持续这种状态，直到以下某一种事件发生:

* 赢得选举
* 其他server成为leader
* 一段时间后，选举没有获胜者

​        candidate只有在一个term时间内，获得集群中超过半数的投票才可以赢得选举。在指定的term内，一个server最多只能投票一个candidate，由此在一个term内最多只有一个leader产生。一旦一个candidate赢得选举后，它成为leader，它发心跳消息给其他server宣示自己的存在并阻止新的选举。

​        第三种可能的结果是，一个candidate既没有赢得选举也没有输掉选举，如果很多followers在同一时间成为candidate，选票会很分散导致没有人赢，然后time out进入下一轮选举，这时如果没有额外干涉的话，选票仍然会很分散，然后继续timeout，无限循环下去。

​        Raft使用随机选举timeout确保split votes很少且能很快被解决。为了防止split vote第一次发生，election timeout从一个固定区间随机产生(比如 150-300ms)。这种设置通常只有一个server timeout，然后它赢得选举成为leader。这种机制也能处理split vote，每个candidate在各自的随机时间后才重新开启新一轮选举，避免split vote无限循环下去。

#### 5.3 Log replication

​        当leader产生后，它开始服务用户请求，每个用户请求包含了一个需要被状态机执行的命令。leader将命令追加到自己的log中国，然后发送 AppendEntries RPC给其他server让它们也记录该请求。当该log entry被safely replicated，leader会执行该命令然后返回结果给client。如果followers crash或者run slowly或者网络丢包，leader会无限重试AppendEntries直到所有follower最终都记录了所有的log。

​        

![image-20211202153424388](/assets/2021/12/raft/image-20211202153424388.png)

​        Logs的组织结构如图六。每个log entry都包含了命令，以及entry被leader创建时的term编号。log中的term编号是为了检测logs中的不一致。每个log entry还有一个index表示其在log中的位置。

​		Leader决定何时执行log entry中的命令是安全的，安全的log entry被称为committed。Raft保证committed entries被持久化，且最终会被所有可用的状态机执执行。一个log entry有大多数副本时，就认为是committed。同时之前的log entry也被认为是committed。leader会一直维护它所知道的最大的committed index，并将这个信息通过AppendEntries RPC通知给其他server，其他server可以根据这个信息在自己的状态机上执行已经committed的命令

​		Raft维护以下特性

* 在两个不同的logs中，相同index和term的log entry应当包含同样的命令
* 在两个不同的logs中，相同index和term的log entry，它们之前的log也应当相同

第一条是自然的；第二条通过AppendEntries时的小校验来实现。当leader有新的log要发送给server时，它也会把新log的前一条log的index和term number发给server。如果server在自己的logs中没有同时满足index和term number的log entry，那么server会拒绝写入新的log entry。

​		在正常执行情况下，leader和follower的日志是一致的，因此AppendEntries的一致性校验不会失败。然而，leader宕机会导致出现不一致，因为旧leader中还有一些log entry没有committed。这种不一致会因为一系列leader和follower的宕机而组成。

![image-20211202160723954](/assets/2021/12/raft/image-20211202160723954.png)

图七显示了可能出现的不一致，follower中可能缺少leader中的log，也可能多。多是因为follower之前可能是leader，当时接受了client request但还没来得及完全replication就宕机了。

​        Raft让leader处理这种不一致，leader会强迫follower更新log，使其同步到leader自己的log内容。同步的方式也很简单，leader为每一个follower维护一个各自的nextIndex，表示nextIndex之前的log entry已经一致，leader刚掌权时，nextIndex就是当前最后一个index+1，比如图7中的11。然后leader和每一个follower通信检测nextIndex是否满足，如果不满足，就nextIndex-1，直到找到真正的nextIndex，然后强行改写follower在nextIndex之后的log entry，并再次更新nextIndex。

#### 5.4 Safety

目前已经讨论了leader选举和复制log entry，但现有机制还不足以保证所有状态机按同样的顺序执行同样的命令。比如一个follower可能在leader commit log entry时宕机了，那么它就没有执行log entry中的命令。当它恢复后可能成为leader，然后重写这些log，不同的机器就可能执行不同的命令。

本节将完善Raft算法，方法是添加成为leader的限制，限制是leader必须拥有之前所有committed log。

##### 5.4.1 Election restriction

​        Raft在投票阶段，candidate发起的RequestVote RPC中包含自己的log信息，candidate必须保证自己的log比大多数server更新更全。如果server的log比candidate的log更up to date，那么server不会给candidate投票。

​        Raft通过比较log中最后一个entry的index和term来决定两个log谁更up-to-date，如果两个log最后的term number不一样，那么term number大的更up-to-date；如果一样，那随便哪个都可以是更大

#### 5.4.2 Committing entries from previous terms

​		![image-20211202175721799](/assets/2021/12/raft/image-20211202175721799.png)

​        根据之前的描述，leader知道当一个log entry被多数server存储后，entry就是committed。但如果一个leader在commit 一个entry之前就宕机，之后的leader将尝试结束replicate the entry。然而，一个leader不能立即确认一个存储在多数server中的entry是否已经committed。图8描述了一种场景，旧的log entry被存储在多数server，但仍然可能被之后的leader重写。

​        为了消除图8中的问题，raft从不通过对entry计数的方式来commit之前term的entry。只有当前term的entry才通过计数的方式来commit。一旦当前term的entry被commit了，那么可以间接推导出之前的也commit了，因为有Log Matching Property。有些情况下leader也能确认一个旧entry是否已经被commit，比如entry存在所有server中，但Raft为了简单使用一种更保守的方法。

#### 5.4.3 Safety argument

