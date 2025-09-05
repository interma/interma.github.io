---
layout: page
title: Postgres
permalink: /postgres/
---

## 我自己的一些关于Postgres/Greenplum的记录

## => Postgres基础
### 参考书
目前使用AI来解读pg代码已经非常强大了，除了代码之外，推荐的几本书籍：
* Hironobu's [PG-JP book](https://www.interdb.jp/pg/)
* Egor's [PG14 book](https://postgrespro.com/community/books/internals)
* 树杰的2本技术内幕：事务处理和查询优化

### 存储
* [存储文件和格式]({% post_url 2025-09-05-storage-format %})
* SMGR
* Bufferpool
* 后台写进程
* Heap
* Index
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



## => Greenplum思考
### 架构再思考

### TP上的提升

### 更好的列存

### 执行器架构和Interconnect
