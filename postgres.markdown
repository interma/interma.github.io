---
layout: page
title: PG文章
permalink: /postgres/
nav_order: 1
---
个人关于Postgres/Greenplum的一些记录

## Postgres内部实现简介
以极简方式对pg的一些内部实现进行介绍。
> 目前已经有很多分析pg内部实现的资料了，为什么你又要写？算是我之前的学习备忘吧，另外毕竟千人千面，可能这里的介绍恰好对你有帮助

### 参考书
目前使用AI来解读pg代码已经非常强大了，除了代码之外，推荐的几本书籍：
* Hironobu's [PG-JP book](https://www.interdb.jp/pg/)
* Egor's [PG14 book](https://postgrespro.com/community/books/internals)
* 树杰著的2本postgres技术内幕：[事务处理](https://book.douban.com/subject/35543446/)和[查询优化](https://book.douban.com/subject/30256561/)

### 存储
* ✔[存储文件和格式]({% post_url 2025-09-05-storage-format %})
* ✔[SMGR]({% post_url 2025-09-09-smgr %})
* Bufferpool
* 后台写进程
* TableAM和Heap操作
* ✔[Btree索引]({% post_url 2025-09-28-btree %})
* 系统表和syscache/relcache

### Xlog
* xlog基础
* 作为复制流的xlog

### 锁
* ✔[常规锁]({% post_url 2025-10-09-relation_lock %})
* ✔[lw锁，行锁和其他]({% post_url 2025-10-10-other_lock %})

### 事务
* ✔[MVCC和快照]({% post_url 2025-10-17-mvcc %})

### 优化器
* ✔[优化器基础和资料汇总]({% post_url 2025-10-01-planner %})

### 执行器
* ✔[执行器基础]({% post_url 2025-09-16-executor-common %})
* Scan家族
* 常用算子
* 写操作Insert/Update

### 其他
* 对pg进行扩展

## Greenplum相关
### 基础
同事们写的GP简介：《如何将postgres做成一个分布式MPP数据库》 
[第一部分](https://www.infoq.cn/article/3IJ7L8HVR2MXhqaqI2RA)，[第二部分](https://www.infoq.cn/article/iadfebtb1y0mojlvrscu)

### 思考
自己gp开发中的一些思考：
* 架构再思考
* gp在OLTP上的性能提升
* 更好的列存实现
* 执行器架构和interconnect

### 其他
之前写给GP社区的几篇文章（与PG实现基本一致）
* [gp代码中的原子操作](https://blog.csdn.net/gp_community/article/details/124636303)
* [gp中的快照](https://blog.csdn.net/chrisy521/article/details/122590844)
* [gp中的复合索引](https://juejin.cn/post/6876618512350216205)
