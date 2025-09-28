---
layout: page
title: PG文章
permalink: /postgres/
nav_order: 1
---
个人关于Postgres/Greenplum的一些记录

## Postgres内部实现简介
以极简方式对pg内部的一些实现进行介绍。
> 目前已经有很多分析pg内部实现的资料了，为什么你又要写？算是我之前的学习备忘吧，另外毕竟千人千面，可能这里的介绍恰好对你有帮助

### 参考书
目前使用AI来解读pg代码已经非常强大了，除了代码之外，推荐的几本书籍：
* Hironobu's [PG-JP book](https://www.interdb.jp/pg/)
* Egor's [PG14 book](https://postgrespro.com/community/books/internals)
* 树杰著的2本技术内幕：事务处理和查询优化

### 存储
* ✔[存储文件和格式]({% post_url 2025-09-05-storage-format %})
* ✔[SMGR]({% post_url 2025-09-09-smgr %})
* Bufferpool
* 后台写进程
* TableAM和Heap操作
* [Btree索引]({% post_url 2025-09-28-btree %})
* 系统表和syscache/relcache

### Xlog

### 锁

### 事务
* MVCC和快照

### 优化器

### 执行器
* ✔[执行器基础]({% post_url 2025-09-16-executor-common %})
* Scan家族
* Insert/Update
* 常用算子

### 其他
* 对postgres进行扩展

## Greenplum相关
### 基础
同事们写的GP简介：《如何将postgres做成一个分布式MPP数据库》 
[第一部分](https://www.infoq.cn/article/3IJ7L8HVR2MXhqaqI2RA)，[第二部分](https://www.infoq.cn/article/iadfebtb1y0mojlvrscu)

### 思考
自己gp开发中的一些思考：
* 架构再思考
* OLTP上的性能提升
* 更好的列存实现
* 执行器架构和interconnect

### 其他
一些小型主题
* ✔[gp代码中的原子操作](https://blog.csdn.net/gp_community/article/details/124636303)（之前写给gp社区的）
* ✔[gp中的快照](https://blog.csdn.net/chrisy521/article/details/122590844)（之前写给gp社区的）
