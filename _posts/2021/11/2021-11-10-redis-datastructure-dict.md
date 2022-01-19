---
layout: post
title:  "Redis v6.0.2 数据结构-dict"
date:   2021-11-10 21:34:01 +0800
categories: Redis Redis数据结构
---



在dict.h中，首先定义了键值对数据结构dictEntry，每一项包含一个key和value，同时有指向下一项的指针

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

多个entry即可组成一个哈希表dictht

```c
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size; //指table的长度
    unsigned long sizemask; //总是为size - 1
    unsigned long used; //指table中实际有多少个entry
} dictht;
```

将哈希表进一步封装成为字典dict

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; //注意，这里是有两个哈希表，是因为rehash会用到
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;
```



### 1. 创建dict

需要指定type和privdata，rehashidx为-1，其余项为0或NULL



### 2. 插入数据

插入数据的核心思想是先创建一个只有key的entry，然后再将value写入entry。如果key已经存在，则返回的entry为NULL.

创建entry需要先根据key的值计算出entry应当在table数组中的位置index，然后创建entry，并将新entry的next指针指向原来的table[index]。在计算index的过程中，可能需要对哈希表进行扩容。注意，哈希表的哈希操作是由创建字典时提供的type决定，这样实现了多态。

查找index的方法是将key的哈希值和sizemask做按位与，但需要检查table[index]这个链表中是否已经存在key

### 3. 扩容

在插入新的key-value时，如果检测到当前哈希表状态需要扩容，则会在ht[1]中分配足够多的空间，并设置dict->rehashidx = 0。但此时并不复制ht[0]中entry的内容到ht[1]中，检测key是否存在仍然先在ht[0]中判断，然后再去ht[1]中判断，最后返回的entry是ht[1]中的entry。这样做的好处是在插入时只标记正在扩容，而不进行耗时的复制操作，提高系统的吞吐量。



如果在插入时发现哈希表已经被标记在rehash中，则会执行Rehash操作，rehashidx代表了本次Rehash操作应当操作哪一个链表，将链表的所有项全部重新哈希到ht[1]中。这样相当于哈希表"慢慢地"从执行扩容操作，将长时间的工作均摊到每一次插入中，避免用户长时间得不到响应。



如果在插入时哈希表正处于扩容状态中，则不会计算哈希表的状态去判断其是否需要再次扩容。每次扩容需要等上一次结束才可能开始。

