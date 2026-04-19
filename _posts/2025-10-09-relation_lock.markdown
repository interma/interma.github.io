---
layout: post
title:  "常规锁"
date:   2025-09-04 08:45:18 +0800
permalink: /postgres/relation_lock.html
tags: [postgres, lock]
---

本文关注 PostgreSQL 中常规锁（主要是表锁）的一些要点；至于其他类型的锁（如 LWLock、行锁等），会在[另一篇文章]({% post_url 2025-10-10-other_lock %})中介绍。

### 简介
常规锁在 PG 中也常被称为重量级锁，也就是 `pg_locks` 里对**用户可见**的那部分锁。`locktype` 字段标识锁类型，而每种类型上又定义了不同的 lock mode：
```
ubuntu=*# select locktype,relation,virtualxid,transactionid,pid,mode,granted from pg_locks;
   locktype    | relation | virtualxid | transactionid |  pid  |       mode       | granted
---------------+----------+------------+---------------+-------+------------------+---------
 relation      |    24588 |            |               | 79313 | RowExclusiveLock | t
 virtualxid    |          | 3/12       |               | 79313 | ExclusiveLock    | t
 transactionid |          |            |           770 | 79313 | ExclusiveLock    | t
```

常规锁除了用户可见之外，还具有等待队列、死锁检测、事务结束自动释放等特性。

表锁（`locktype = relation`）是最重要的一类常规锁。出于性能和兼容性考虑，PG 在它上面划分了 8 种锁模式；而其他一些锁类型往往只有经典的共享 / 互斥两种模式。我们经常需要查如下这张互斥表：
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
可以先用一个朴素思路来理解 PG 中常规锁的内部实现：既然锁本身是资源对象，而锁又可能被多个后端进程（`PGPROC`）持有，那就还需要另一个结构来表示“进程和锁之间的关系”。

代码中对应的两个核心结构就是 `LOCK` 和 `PROCLOCK`：前者表示锁对象；后者表示 `PGPROC` 和 `LOCK` 之间的多对多关系。

它们的关系图如下：
```txt
[LOCK] --1:N--> [PROCLOCK] <--N:1-- [PGPROC]  // LOCK和PGPROC的多对多关系
```

几个结构体中的关键字段如下：
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

有了这些结构之后，下一个问题就是如何把它们组织起来。PG 的做法非常直接：使用共享内存中的哈希表，并通过不同 tag 访问它们。
平时我们所说的“锁表”，主要就是下面三个：
- LockMethodLockHash：用于存储锁对象（Lock）的哈希表，用于全局管理数据库中的锁
- LockMethodProcLockHash：用于存储进程锁（ProcLock）的哈希表，跟踪每个进程持有的锁
- LockMethodLocalHash：用于存储本地锁信息的哈希表，管理当前进程视角下的锁状态，主要起加速作用

三者协同工作，构成了pg在并发控制中高效且细粒度的锁管理基础设施。

### 锁操作
加锁函数 `LockAcquire()` 的详细逻辑可以参考树杰的书。无锁冲突时，大致流程如下：
1. 在 LockHash 中查找所需的 `LOCK`，不存在就创建。
2. 在 ProcLockHash 中插入 `(PGPROC, LOCK)` 对应的 `PROCLOCK`。
3. 更新 `LOCK` 和 `PROCLOCK` 中的状态，例如持有模式、等待信息等。
4. 事务结束时释放 `PROCLOCK`；如果某个 `LOCK` 已不再被任何 `PROCLOCK` 引用，则进一步释放该 `LOCK`。

如果在第 1 步发现锁冲突，当前进程就要进入等待，并在其他进程释放锁后再被唤醒。
这两部分的关键逻辑如下：

★锁等待逻辑：函数链 `LockAcquire() -> WaitOnLock() -> ProcSleep()`，最终在 `ProcSleep()` 中完成：
1. 检查冲突。如果没有冲突，就直接获取锁返回；如果有冲突，还要决定自己是插到队尾，还是插到某个冲突者之前。
2. 做死锁检测。如果发现自己和队列中的某个进程形成无法通过调序解决的依赖环，就会设置 `early_deadlock` 标志。
3. 插入等待队列。当前进程会被插入等待队列中的合适位置，同时更新 `LOCK` 和 `PGPROC` 中与 `waitLock` 相关的字段。
   > 等待队列实际就是：`dclist_head *waitQueue = &lock->waitProcs;`
4. 进入等待循环。在 `WaitLatch` 上休眠并等待被唤醒。
5. 唤醒后检查是否已经拿到锁、是否超时或死锁，然后视情况记录日志并重试。

★锁唤醒逻辑：事务结束时，会沿着 `ProcReleaseLocks() -> LockReleaseAll() -> CleanUpLock() -> ProcLockWakeup()` 这条路径唤醒等待进程。在 `ProcLockWakeup()` 中：
1. 遍历等待队列 `lock->waitProcs` 中的每个 `PGPROC`。
2. 判断其请求的 lock mode 是否与**当前已授予的锁，以及排在它前面的请求**冲突。
3. 如果不冲突，就把它从队列中移除并唤醒；否则继续判断后续进程。
   > 因此一次可能唤醒多个彼此不冲突的等待者，比如一批读请求。

★死锁检测：原理可以简单理解成“判断锁等待图中是否存在依赖环”。树杰的书里讲得很详细。GP 的分布式死锁检测也是同样的思路，只不过需要先收集全局锁信息，可参考[详解 GP 全局死锁检测器](https://juejin.cn/post/6844904192855818253)。

### Discussion
对于任何服务端软件，锁机制都是并发控制的一种重要手段，同时也是性能瓶颈之一，需要精妙设计。

PostgreSQL 中锁的内容很多，而且相当偏细节。更多参考资料：
- 外部行为：可以看 PG14 book，以及这篇 [Everything You Need To Know About PostgreSQL Locks: Practical Skills You Need](https://mohitmishra786.github.io/chessman/2025/03/02/Everything-You-Need-to-Know-About-PostgreSQL-Locks-Practical-Skills-You-Need.html)。
- 内部实现：可以参考树杰《事务处理》一书中的锁章节。

另外还可以思考一下锁与同为并发控制机制的MVCC之间的区别与互补。

Questions:
1. 系统函数 `pg_blocking_pids(pid)` 会返回阻塞指定 pid 的所有其他进程。思考一下它是如何实现的？
    > 主要是"软"阻塞情况该如何处理
2. 代码中打开表要加特定模式：`relation_open(rel, mode)`；为什么随后的关闭表操作往往直接用 `NoLock`：`relation_close(rel, NoLock)`？
3. 直观上看，lock 和 proc 似乎应该总是通过 proclock 建立关系；但代码里还有不少 lock 和 proc 直接打交道的地方，比如 `PGPROC.waitLock` 保存当前进程正在等待的锁，`LOCK.waitProcs` 保存等待队列。为什么要这么设计？
    > 涉及到代码细节，可以询问一下chatgpt
