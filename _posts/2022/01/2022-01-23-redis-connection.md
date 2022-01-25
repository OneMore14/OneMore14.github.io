---
layout: post
title:  "Redis v6.2.0 server接收client命令"
date:   2022-01-23 14:51:01 +0800
categories: Redis 
---

以standalone模式按默认配置启动redis-server，只考虑最基本的情况

## 1. 准备连接

```c
struct redisServer {
    int ipfd[CONFIG_BINDADDR_MAX]; /* TCP socket file descriptors */
    int ipfd_count;                /* Used slots in ipfd[] */
} 
```

默认ipfd_count为2，一个监听本地TCP4，另一个监听本地TCP6。