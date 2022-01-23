---
layout: post
title:  "Redis v6.2.0 数据结构-list"
date:   2021-11-10 10:39:01 +0800
categories: Redis Redis数据结构
---

Redis中有多种不同的List，将分别介绍

## 1. ADLIST
adlist是普通的链表，由下面3种结构体实现

```c
/* Node, List, and Iterator are the only data structures used currently. */

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

这里的list就是一个双端链表



## 2. ZIPLIST

ZIPLIST是一种空间紧凑的线性结构，它需要占用一段连续的内存，且List中各个entry是前后相接的，因此内存消耗很少。


### ziplistEntry

ziplistEntry可以保存一段字符串，或者保存一个long long整数

```c
/* Each entry in the ziplist is either a string or an integer. */
typedef struct {
    /* When string is used, it is provided with the length (slen). */
    unsigned char *sval;
    unsigned int slen;
    /* When integer is used, 'sval' is NULL, and lval holds the value. */
    long long lval;
} ziplistEntry;
```

### 创建一个空的ziplist

空的ziplist不包含任何entry，只有header和end两部分结构。

```c
/* The size of a ziplist header: two 32 bit integers for the total
 * bytes count and last item offset. One 16 bit integer for the number
 * of items field. */
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
```

```c
/* Size of the "end of ziplist" entry. Just one byte. */
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))
```

明白header和end的结构以后，初始化代码就很比较好理解了

```c
/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

### 合并两个ziplist

首先选出两个list中更大的那一个作为合并后的返回值，称为target。
将target扩容，使其能装下另一个list。再复制另一个list的内容到扩容部分的内存。然后设置target的header和end，并释放原来较小list的内存。

具体过程可以直接看Redis源码，注释很多。



## 3. SKIPLIST

跳表实际是为了zset而存在，也就是有序集合。在Redis中，有序集合是string类型元素的集合，且每个元素关联了一个double值用来比较元素之间大小。元素的字符串本身不能重复，但关联的分数值可以重复。