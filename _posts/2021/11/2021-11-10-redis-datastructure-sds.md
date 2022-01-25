---
layout: post
title:  "Redis v6.2.0 数据结构-字符串sds"
date:   2021-11-10 10:39:01 +0800
categories: Redis Redis数据结构
---

Redis由C语言编写而成，在C语言中，字符串由一个以0结尾的char数组来表示，不易管理。
Redis中以结构体的方式封装了char数组

其结构定义如下

```c
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```