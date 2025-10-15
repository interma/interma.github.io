---
layout: post
title:  "常规锁"
date:   2025-09-04 08:45:18 +0800
permalink: /postgres/relation_lock.html
tags: [postgres, lock]
---

本文关注postgres中常规锁（主要是表锁）的一些要点，而对于其他的锁类型（如lwlock，行锁等）[另起一篇]({% post_url 2025-10-10-other_lock %})介绍。

### 简介
常规锁在pg中又叫重量级锁，即```pg_locks```中对**用户可见**的锁，其中的locktype字段标识了锁的类型，每种类型上有不同的锁模式mode：
```
ubuntu=*# select locktype,relation,virtualxid,transactionid,pid,mode,granted from pg_locks;
   locktype    | relation | virtualxid | transactionid |  pid  |       mode       | granted
---------------+----------+------------+---------------+-------+------------------+---------
 relation      |    24588 |            |               | 79313 | RowExclusiveLock | t
 virtualxid    |          | 3/12       |               | 79313 | ExclusiveLock    | t
 transactionid |          |            |           770 | 79313 | ExclusiveLock    | t
```

常规锁除了用户可见之外，还享受等待队列，死锁检测，自动释放（在事务结束时）等诸多特性。

表锁（```locktype==relation```）是最重要的一种常规锁，为了性能考虑，其上划分了8种锁模式（其他锁一般只有经典的共享/互斥2种模式）。我们经常需要查阅如下的表锁互斥表：
```
表锁互斥矩阵->chatgpt
----
Requested \ Existing ->   AS   RS   RE  SUE    S   SRX   EX   AX   Scenarios
-----------------------------------------------------------------
ACCESS SHARE (AS)        [ ]  [ ]  [ ]  [ ]   [ ]  [ ]  [ ]   X   # SELECT
ROW SHARE (RS)           [ ]  [ ]  [ ]  [ ]   [ ]  [ ]   X    X   # SELECT ... FOR SHARE
ROW EXCLUSIVE (RE)       [ ]  [ ]  [ ]  [ ]    X    X    X    X   # INSERT, UPDATE, DELETE, SELECT ... FOR UPDATE
SHARE UPDATE EXCL (SUE)  [ ]  [ ]  [ ]   X     X    X    X    X   # VACUUM, ANALYZE, CIC
SHARE (S)                [ ]  [ ]   X    X     X    X    X    X   # CREATE INDEX
SHARE ROW EXCL (SRX)     [ ]  [ ]   X    X     X    X    X    X   # CREATE TRIGGER, CREATE RULE
EXCLUSIVE (EX)           [ ]   X    X    X     X    X    X    X   # REFRESH MATERIALIZED VIEW
ACCESS EXCLUSIVE (AX)     X    X    X    X     X    X    X    X   # DROP TABLE, VACUUM FULL, ALTER TABLE (DDL)
```

### 共享内存中的锁表
我们可以用朴素的思路来思考pg中常规锁的内部实现：首先要有作为资源的锁对象，再考虑到锁可以被多个后端进程（`PGPROC`）持有，因此还需要引入另一个对象来保存它们的对应关系。

对应代码中的这2个结构就是`LOCK`和`PROCLOCK`：前者表示锁对象；而后者表示了`PGPROC`和`LOCK`的多对多关系。

它们的关系图如下：
```txt
[LOCK] --1:N--> [PROCLOCK] <--N:1-- [PGPROC]  // LOCK和PGPROC的多对多关系
```

相关结构体中的关键字段：
```c
struct Lock
    LOCKTAG tag             // locktag: 不同的资源不一样，比如relation oid
    dlist_head procLocks    // 双向链表，存放所有与该锁关联的PROCLOCK对象
    dclist_head waitProcs   // (带计数的)双向链表，存放所有正在等待该锁的PGPROC对象，即我们常说的常规锁的等待队列
    ... // 其他字段：请求和持有的lockmask等

struct PROCLOCK
    PROCLOCKTAG tag         // proclock tag: lock指针+proclock指针
    dlist_node lockLink     // 作为链表节点，挂载到对应LOCK的proclocks链表中，便于查找所有持有某锁的进程
    dlist_node procLink     // 作为链表节点，挂载到对应PGPROC的proclocks链表中，便于查找某进程持有的所有锁
    ... // 其他字段：lockmask和group leader等成员

struct PGPROC
    LOCK *waitLock          // 当进程因为锁冲突而进入休眠状态时，指向它具体在等待哪个锁
    PROCLOCK *waitProcLock	// 所等待锁的PROCLOCK
    dlist_head	myProcLocks[NUM_LOCK_PARTITIONS]; // 分区链表，保存了本进程所持有或等待的所有PROCLOCK
    ...
```

有了这些结构之后，接下来需要考虑如何将他们组织到一起：pg的方式非常简明直接，使用了（共享内存中的）哈希表来存储（通过不同的tag来访问）。
我们一般所谈及到的锁表主要就是如下3个：
- LockMethodLockHash：用于存储锁对象（Lock）的哈希表，用于全局管理数据库中的锁
- LockMethodProcLockHash：用于存储进程锁（ProcLock）的哈希表，跟踪每个进程持有的锁
- LockMethodLocalHash：用于存储本地锁信息的哈希表，用于管理当前进程的锁，主要起加速作用

三者协同工作，构成了pg在并发控制中高效且细粒度的锁管理基础设施。

### 锁操作
加锁函数`LockAcquire()`（其详细逻辑见树杰书），无锁冲突时的简要流程是：
1. 在LockHash中查找所需的LOCK，如果没有就创建
1. 在ProcLockHash中插入(PGPROC,LOCK)对应的PROCLOCK
1. 修改LOCK和PROCLOCK中的状态：如持有模式、等待队列等
1. 待事务结束时，释放PROCLOCK，如果LOCK无任何对应的PROCLOCK，则释放LOCK

如果step1处发现锁冲突，那当前进程就要进行锁等待，然后等待step4处其他进程释放锁之后来唤醒它。
这2部分的关键逻辑如下：

★锁等待逻辑：函数链`LockAcquire() → WaitOnLock() → ProcSleep()`，最终在`ProcSleep()`中完成：
1. 检查冲突：
如果没有锁冲突，则直接获取锁并返回。有冲突时，需要判断自己应该插入到队尾还是插入到第一个冲突者之前（场景：这个冲突者将要等待我的锁）
1. 死锁检测：
如果发现自己和队列中的某个进程冲突且无法调序解决，说明存在死锁并设置`early_deadlock`标志
1. 插入等待队列：
将当前进程插入到等待队列的合适位置，更新LOCK和PGPROC对象的`waitLock`相关字段
    > 等待队列实际就是：dclist_head *waitQueue = &lock->waitProcs;
1. 等待循环：
进入主等待循环，等待在`WaitLatch`上所唤醒
1. 唤醒后处理：
被唤醒后，检查是否获得锁、是否超时或死锁等，并根据情况记录详细log，然后重新尝试获得锁

★锁唤醒逻辑：在事务结束时，函数链`ProcReleaseLocks() → LockReleaseAll() → CleanUpLock() → ProcLockWakeup()`来唤醒等待的进程。在`ProcLockWakeup()`中：
1. 遍历等待队列`lock->waitProcs`中的每个PGPROC
1. 判断其请求的lockmode是否与**当前以及其队列前边的其他请求**冲突
1. 如果不冲突，则队列中移除并唤醒；否则继续遍历判断其他进程
    > 因此可以一次唤醒多个不冲突的PGPROC，比如多个读请求的进程

★死锁检测：原理简单说就是判断锁表中是否存在依赖环（见树杰的书中详解）。gp的分布式死锁检测也是同理，只不过需要收集全局的锁信息，[详解GP全局死锁检测器](https://juejin.cn/post/6844904192855818253)。

### Discussion
对于任何服务端软件，锁机制都是并发控制的一种重要手段，同时也是性能瓶颈之一，需要精妙设计。

postgres中锁的内容较多，且偏细节，更多参考：
- 外部行为：可以参考PG14book和这篇[Everything You Need To Know About Postgresql Locks: Practical Skills You Need](https://mohitmishra786.github.io/chessman/2025/03/02/Everything-You-Need-to-Know-About-PostgreSQL-Locks-Practical-Skills-You-Need.html)
- 内部实现：可以参考树杰的《事务处理》一书中的锁章节

另外还可以思考一下锁与同为并发控制机制的MVCC之间的区别与互补。

Questions:
1. 系统函数`pg_blocking_pids(pid)`：返回阻塞指定pid进程的所有其他进程的pid列表。思考一下它是如何实现的？
    > 主要是"软"阻塞情况该如何处理
1. 代码中打开表需要加上特定的模式：`relation_open(rel, mode)`，为什么随后的关闭表操作往往直接采用了nolock模式：`relation_close(rel, NoLock)`？
1. 直观感觉lock和proc一定要通过proclock建立关系，但是代码中还有很多地方是lock和proc之间打交道的，比如：`PGPROC.waitLock`保存当前进程所等待的锁；`LOCK.waitProcs`保存的PGPROC等待列表，为什么这么设计？
    > 涉及到代码细节，可以询问一下chatgpt