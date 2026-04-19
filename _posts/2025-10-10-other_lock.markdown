---
layout: post
title:  "其他锁"
date:   2025-09-07 13:24:12 +0800
permalink: /postgres/other_lock.html
tags: [postgres, lock]
---

上一篇已经介绍了[常规锁]({% post_url 2025-10-09-relation_lock %})，这篇文章继续补充 PG 中其他几类常见锁。

### spinlock
spinlock 即自旋锁，PG 内部直接通过 CPU 的 TAS/CAS 指令实现。

因此它只有互斥这一种模式。典型用法如下，用来原子性地修改多个变量：
```c
SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    CheckpointerShmem->ckpt_failed++;
    CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
SpinLockRelease(&CheckpointerShmem->ckpt_lck);
```

如果只是修改单个变量，通常直接使用 PG 内部的原子操作即可（`src/backend/port/atomics.c`）。我之前也写过一篇介绍：[GP 代码中的原子操作](https://blog.csdn.net/gp_community/article/details/124636303)。其中一部分原子操作在底层也是借助 spinlock 实现的。

### lwlock
LWLock 即 lightweight lock，作用**类似编程语言里的 mutex**。它对用户不可见，也不像常规锁那样自带死锁检测和事务结束自动释放机制，开发者必须在代码中手动获取和释放 LWLock，并自行保证正确性。

代码中的实现结构是：
```c
typedef struct LWLock
{
    uint16 tranche;         /* tranche ID*/	//->锁类别
    pg_atomic_uint32 state; /* state of exclusive/nonexclusive lockers */
    proclist_head waiters;  /* list of waiting PGPROCs */ //->也有等待队列
} LWLock;
```

在 `src/include/storage/lwlocklist.h` 中可以看到一系列预定义的 LWLock。

其内部实现细节仍然推荐参考树杰《事务处理》一书中的锁章节；而一个很典型的使用场景，则是[buffer pool 中的并发控制](https://zhmin.github.io/posts/postgresql-buffer-concurrent/)。

### row lock
为了实现更细粒度的并发控制，必须引入行锁。它是 PG 中非常重要的一类锁，其实现依赖 tuple header 里的 `xmax + infomask` 标记。PG 在行锁上定义了 4 种模式，本质上是在共享 / 互斥的基础上，再区分是否涉及 key update。常见使用场景如下：
```
begin; select ... FOR [KEY] SHARE;      // 直接申请 FOR 后写明的模式
begin; select ... FOR [NOKEY] UPDATE;   // 同上
update ... ;                            // 通常是 FOR NO KEY UPDATE；若修改了可被外键引用的唯一索引键列，则会提升到 FOR UPDATE
delete ... ;                            // FOR UPDATE
insert ... ;                            // 一般不需要显式行锁，主要依赖 MVCC
select ... ;                            // 普通 SELECT 不加行锁，主要依赖 MVCC
```

行锁的加锁过程并不只是“给 tuple 打一个标记”，而是还需要 tuple lock 的配合。简化后的过程大致如下：
1. 如果 `xmax` 和 hint bits 表明该行当前正被排他方式锁住，先获取一个重量级的 **tuple lock**。
2. 如有需要，再去等待当前持锁事务释放冲突锁；如果 `xmax` 中存的是 multixact，也可能需要等待多个事务。
3. 把自己的事务 ID 写入 tuple header 中的 `xmax`，并设置对应的标记位。
4. 如果第 1 步获取了 tuple lock，这里再把它释放掉。
   > tuple lock 本质上是一种常规锁。引入它的重要目的之一，是识别和保护“第一个等待者”，避免长时间饥饿。

更简洁的伪代码表示：
```c
LockTuple()           // acquire tuple lock
XactLockTableWait()   // wait on another txn (if required)
mark tuple            // set xmax + infomask => locked by me
UnlockTuple()         // release tuple lock
```

另外，PG 中很多死锁和性能问题都和行锁有关；实现细节可以结合 PG14 book 中关于行锁的章节一起看。

### others
除此之外，还有 object lock、advisory lock、page lock 等更细分的类型。PG14 book 里专门有一章介绍它们，可以按需查阅。

Questions:
1. 对于 `PageLock`（见 `LockPage()` 函数），直观上似乎每次访问 page / buffer 都该用它保护，形成“表锁 -> page 锁 -> 行锁”的层次；但实际真是这样吗？
    > 实际并不是。就 PostgreSQL 主线代码里常见的使用场景来看，`PageLock` 主要出现在 GIN 这类少数路径中；而 buffer pool 则是通过 LWLock 来保护的。
