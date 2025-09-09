---
layout: page
title: PG文章
permalink: /postgres/
nav_order: 1
---

个人关于Postgres/Greenplum的一些记录

## Postgres点滴
### 参考书
目前使用AI来解读pg代码已经非常强大了，除了代码之外，推荐的几本书籍：
* Hironobu's [PG-JP book](https://www.interdb.jp/pg/)
* Egor's [PG14 book](https://postgrespro.com/community/books/internals)
* 树杰著的2本技术内幕：事务处理和查询优化

### 存储
* ✔[存储文件和格式]({% post_url 2025-09-05-storage-format %})
* [SMGR]({% post_url 2025-09-09-smgr %})
* Bufferpool
* 后台写进程
* Heap表操作
* 索引
* Syscache/Relcache

### Xlog

### 锁

### 事务
* MVCC和快照

### 优化器

### 执行器
* 执行器基础
* Scan家族
* Insert/Update
* 常用算子

### 其他
* 扩展Postgres

## Greenplum思考
* 架构再思考
* OLTP上的性能提升
* 更好的列存实现
* 执行器架构和Interconnect
