<!-- ---
layout: post
title:  "MapReduce 论文阅读"
date:   2021-11-22 20:34:01 +0800
categories: 6.824
--- -->

---
layout: post
title:  "6.S081 xv6 sbook chapter1: Operating system interfaces"
date:   2021-11-22 21:34:01 +0800
categories: 6.S0s81
---



# MapReduce: Simplified Data Processing on Large Clusters

### 1 Introduction



### 2 Programming Model

MapReduce是一种编程模型，输入key/value pairs，输出也是key/value pairs。 用户需要自定义Map和Reduce两个操作

* Map 输入原始数据的键值对，产生一系列键值对形式的中间结果，将这些中间结果根据key值聚集起来，传递给Reduce操作
* Reduce 接受中间结果，对同一个key下的value进行合并操作，最终得到0或1个结果

### 2.1 Example

以一个统计文本中单词出现次数的代码为例

```java
map(String key, String value):
	// key: document name
	// value: document contents
	for each word w in value:
		EmitIntermediate(w, "1");
```

```java
reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

### 2.2 Types

MapReduce的入参和结果类型需要由用户自行定义与字符串的转换方式

```
map  (k1,v1) → list(k2,v2)
reduce  (k2,list(v2)) → list(v2)
```

### 2.3 More Examples

* Distributed Grep
* Count of URL Access Frequency
* Inverted Index
* Distributed Sort



### 3 Implementation

MapReduce可以根据具体情况有不同的实现，论文作者介绍了在Google的硬件环境

* 双处理器x86计算机，内存2~4Gb，Linux操作系统
* 100Mbit/s或者1Gbit/s的以太网
* 几百上千台计算机的集群，因此机器故障很常见
* 存储是普通磁盘直接挂载在计算机上，由分布式文件系统管理
* 用户提交jobs给调度系统，每个job由一套task组成，并被调度系统分配给集群中一套可用的机器





![image-20211122211703002](/assets/2021/11/mapreduce/image-20211122211703002.png)

### 3.1 Execution Overview

Map调用被自动分散在不同的机器上，输入可以在不同机器上并行处理，Reduce需要根据Key的值做partition决定分配。

1. 在用户程序中的MapReduce库首先将输入文件分成M份，每份大小一般是16M到64M，由配置参数决定。然后在集群机器上启动许多程序的副本
2. 有一份特殊的副本是master，其余是worker，worker的任务由master分配，一共有M个map task和R个reduce task，由master为空闲worker分配任务，任务类型为map或reduce二选一
3. map类型的worker读取分到的input data，将其转换为key/value pairs，再将这些pairs传递给map函数，map函数产生的中间结果缓存在内存中
4. 缓存的中间结果周期性写入硬盘中，按规则分成R个分区，这些中间结果键值对在硬盘的位置被传回给master节点，master节点将转发这些位置给reduce worker。
5. master告知reduce worker中间结果的位置后，reduce worker通过RPC调用读取对应的数据，然后根据key对pairs排序，以便于将相同key的value聚集在一起。如果数据量太大，将使用外部排序。
6. reduce遍历数据，并将key和对应的value传给reduce函数，将reduce返回的结果追加在最终输出的文件中。
7. 当所有map和reduce任务结束后，master唤起用户程序，这时MapReduce函数返回到用户代码。

在成功执行完成后，MapReduce结果存储在R份输出文件中。用户程序往往不需要将R份文件汇聚在一起，而是将它们作为另一次MapReduce的输入，或者作为其他分布式应用的输入。

### 3.2 Master Data Structures

维护worker的状态，中间结果的位置和大小等数据

### 3.3 Fault Tolerance

#### Worker Failure

master会对worker进行心跳检测。map task完成后，将重新回到idle状态，准备调度到其他worker。如果map/reduce task运行时机器故障，那么该任务将被重新调度。已完成的map task如果机器故障，需要重新计算map函数，因为中间结果存储在本地磁盘，机器故障后无法读取；而已完成的reduce task如果机器故障则不需要重新计算，因为最终结果是写在全局文件系统的。当map task重做时，将通知所有reduce worker。

#### Master Failure

master可以周期性将状态写入checkpoint，master故障时可以从最近的checkpoint恢复。单节点master一般不会故障，因此也可以故障后直接放弃任务，由用户代码检测到异常返回并重启MapReduce调用。

#### Semantics in the Presence of Failures

如果map和reduce操作是deterministic functions of their input values，那么整个分布式系统看上去和不出错的顺序执行结果一样。map和reduce的结果都是原子提交，它们都先将输出写入到private temporary files中，map完成后将结果位置告知master，如果master中已经有对应部分的结果，则忽略，否则加入到维护的状态中。reduce则是将临时文件改名，因为reduce是将输出写在分布式文件系统的，因此改名时只有一次成功的机会。

### 3.4 Locality

网络带宽是相对宝贵的资源，为节省带宽，将input files以GFS的形式保存在各机器的本地磁盘。master在分配map任务时，优先选择本地磁盘上已经有对应input的worker或者是离得近的worker。

### 3.5 Task Granularity

我们把map task分成M份，reduce task分成R份，理想情况下，M和R的数量远远超过机器worker的数量，有助于动态负载均衡，也提高worker失败时的恢复速度。通常M需要让每一份数据大小在16M到64M之间，R则是worker的小倍数

### 3.6 Backup Tasks

MapReduce的整体时间可能被个别有问题的机器拖累，为了解决这个问题，当MapReduce快要结束时，如果这时候还有剩余的task在运行，master就会安排新的worker也来做这些task，最后不管是原来的worker先完成还是后安排的worker先完成，对应的task都被标记完成。



### 4 Refinements

### 4.1 Partitioning Function

对于reduce分区，提供默认的分区方式$hash(key)\ mod\ R$ 

### 4.2 Ordering Guarantees

对于给定的分区，保证按中间结果key的升序来调用reduce

### 4.3 Combiner Function

有时候map操作会产生大量的重复结果，而这些中间结果对于reduce来说是commutative和associative的，例如word count任务，map可能产生大量的<"the", 1>结果，这时可以在map任务加一个可选的combiner function，汇聚产生一个<"the", 100>的中间结果，这样减少了网络带宽使用

### 4.4 Input and Output Types

### 4.5 Side-effects

 

## Lecture Notes

