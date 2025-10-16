---
layout: post
title:  "其他锁"
date:   2025-09-07 13:24:12 +0800
permalink: /postgres/other_lock.html
tags: [postgres, lock]
---

之前的文章中已经介绍了[常规锁]({% post_url 2025-10-09-relation_lock %})，这篇文章继续介绍pg中的其他几种锁类型。

### spinlock
即自旋锁，pg内部通过CPU的TAS/CAS指令来实现。

它只有互斥模式，典型示例（原子性修改多个变量）：
```c
SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    CheckpointerShmem->ckpt_failed++;
    CheckpointerShmem->ckpt_done = CheckpointerShmem->ckpt_started;
SpinLockRelease(&CheckpointerShmem->ckpt_lck);
```

另外如果修改单个变量，一般用pg内部的原子操作函数即可（`src/backend/port/atomics.c`），之前写过的一篇介绍[gp代码中的原子操作](https://blog.csdn.net/gp_community/article/details/124636303)。其中部分原子操作函数也是通过spinlock来实现的。

### lwlock
lwlock == lightweight lock，即轻量级锁，其作用非常**类似编程语言中的mutex**。对用户不可见，也不像常规锁那样有死锁检测和自动释放机制，开发者必须在代码中手动申请和释放lwlock，并保证其正确性。

代码中的结构实现是
```c
typedef struct LWLock
{
    uint16 tranche;         /* tranche ID*/	//->锁类别
    pg_atomic_uint32 state;	/* state of exclusive/nonexclusive lockers */
    proclist_head waiters;  /* list of waiting PGPROCs */ //->也有等待队列
} LWLock;
```

在`src/include/storage/lwlocklist.h`中能看到一系列定义好的predefined LWLock list。

其内部实现还是参考树杰的《事务处理》一书中的锁章节，典型的使用case可以参考[bufferpool中的并发控制](https://zhmin.github.io/posts/postgresql-buffer-concurrent/)。

### row lock
行锁是非常重要的一个锁主题，pg中的实现采用了tuple中的xmax+infomask标记。在共享/互斥的基础上再考虑是否存在key update，于是生成4种锁模式。具体的使用场景：
```
begin; select ... FOR [KEY] SHARE;      // 直接申请写明的模式
begin; select ... FOR [NOKEY] UPDATE;   // 同上
update ... ;                            // 根据是否更新[主键/唯一键/外键]来判断NOKEY UPDATE
delete ... ;                            // FOR UPDATE
insert ... ;                            // 一般无行锁，MVCC方式并发控制
select ... ;                            // 无行锁，MVCC方式并发控制
```

pg中很多死锁和性能问题都是行锁导致的，其实现细节可以参考PG14 book。

下边直接摘抄一下行锁的加锁步骤：
1. If the xmax field and hint bits indicate that the row is locked in the exclusive mode,
acquire an exclusive heavyweight **tuple lock**.
2. If required, wait for all the incompatible locks to be released by requesting a lock on
the ID of the xmax transaction (or several transactions if xmax contains a multixact
ID).
3. Write its own ID into xmax in the tuple header and set the required hint bits.
4. Release the **tuple lock** if it was acquired in the first step.
> 留意其中的tuple lock是一种常规锁，引入它的目的是甄别出第一个等待进程，防止其长时间饥饿

更简洁的伪代码表示：
```c
LockTuple()           // acquire tuple lock
XactLockTableWait()   // wait on another txn (if required)
mark tuple            // set xmax + infomask, locked by me
UnlockTuple()         // release tuple lock
```

### others
还有类似object lock，advisory lock等小类型的锁，PG14 book中专门开了一章来写它们，可以参考阅读。

最后留意还有一种PageLock（见`LockPage()`函数），直观感觉像是每次访问page/buffer时使用（层次：表锁->page锁->行锁），而实际上是这样吗？可以结合代码看看。
