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

Redis启动时，通过
```c 
int listenToPort(int port, int *fds, int *count)
```
先创建socket,绑定端口并监听客户端请求。即此时完成了bind()和listen()，但还没有开始accept()

```c
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```
之后的main函数将调用aeCreateFileEvent()为本地监听的socket创建一个可读的文件事件，并将socket加入到server的epoll对象中，并以acceptTcpHandler作为监听socket的回调函数。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```
所有的启动准备工作(启动监听、加载RDB等)结束后，main函数调用aeMain()循环处理所有的文件事件和时间事件

```c
numevents = aeApiPoll(eventLoop, tvp);

for (j = 0; j < numevents; j++) {

    /* Fire the readable event if the call sequence is not
        * inverted. */
    if (!invert && fe->mask & mask & AE_READABLE) {
        fe->rfileProc(eventLoop,fd,fe->clientData,mask);
        fired++;
        fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
    }
}    
```
在aeProcessEvents()中，aeApiPoll()会调用epoll_wait()获取当前已经就绪的事件，然后for循环依次处理，文件可读事件就会调用rfileProc回调函数。

假设这时有客户端发起连接，则for循环中rfileProc实际上就调用了之前注册的acceptTcpHandler函数用于处理连接请求。

## 2. 准备读取 

在acceptTcpHandler中相当于是执行了listen之后的一次accept()，此时已经和client建立了连接但还没有读到client准备执行的命令。这时的情况就类似之前执行了listen()但没有accept()。实际上处理方式也和之前类似，即再为当前的连接设置一个文件读事件，将连接的socket加入到server的epoll对象中，回调函数是readQueryFromClient()，当有客户端命令发来时，readQueryFromClient()就可以读到客户端命令并调用相关的执行函数了。