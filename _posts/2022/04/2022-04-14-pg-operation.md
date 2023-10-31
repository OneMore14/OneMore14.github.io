---
layout: post
title:  "PostgreSql一些记录"
date:   2022-04-14 22:43:01 +0800
tags: 记录
typora-root-url: ../../../
---



为什么老是多



## 操作

查数据库表的大小

![img](https://i.stack.imgur.com/aeous.png)

```sql
select pg_size_pretty(pg_relation_size('table_name'));
```

复制表结构，而且会复制原来表中定义的索引、主键等，但要注意，如果原来的主键是设置了sequence，则会用原来的sequence，而不是新建

```sql
CREATE TABLE mycopy (LIKE mytable INCLUDING ALL);
```

查询所有索引和索引是否有效

```sql
select relname, pg_index.indisvalid from pg_class, pg_index where pg_index.indexrelid = pg_class.oid
```



## 概念

### VACUUM

删除表中行数据后，原有数据占据的磁盘空间并未释放，也不会分配给之后新插入的行。vacuum起到回收磁盘的作用: 标准vacuum将dead tuples占据的空间重新标记为可用，但不会还给操作系统，而是将新数据优先放在原来的空间中，整个过程不需要互斥锁，用户可以正常访问数据库。另一种是full vacuum，会将所有数据完全重写到另一块磁盘空间，然后释放之前所有的空间并还给操作系统，full vacuum速度很慢而且需要互斥锁。官方推荐生产环境数据库执行vacuum，至少是nightly。
