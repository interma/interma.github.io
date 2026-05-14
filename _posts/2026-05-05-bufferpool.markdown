---
layout: post
title:  "PG缓冲池"
date:   2026-05-05 15:19:23 +0800
permalink: /postgres/bufferpool.html
tags: [postgres, bufferpool]
---

PostgreSQL 不是一个内存数据库，如果IO操作直接走磁盘，性能会非常低。因此需要引入一个 bufferpool 来优化性能。

从数据库外部看，buffer pool 有点像一个“超大版 KV cache”：key 是 `(rel, fork, block)` 这样的页标识；value 是对应的 page 内容。

不过，真正的 PostgreSQL buffer pool 远不只是“一个 hash 表 + 一块内存”，至少还要同时解决这些问题：
- 多个 backend 并发访问同一个 page 时的并发控制。
- 脏页的异步（为了性能需要）刷写策略。
- buffer 替换算法。
- 各种性能优化：空间配置；IO操作优化；防止顺序扫描污染；可观测性等等。

### 基础
一个最简单 KV cache，大致可以认为是`hash(key) -> entry -> value`。

而 PG 的 buffer pool 可以简化为如下结构：

```txt
  shared hash table
        |
        |  BufferTag(rel,fork,block) -> buffer_id (slot No.)
        v
 shared buffers[buffer_id]
        |
        +--> BufferDesc: lock / flags / ...
        +--> page data (8KB)
```

其中：
- `BufferTag` 相当于 cache key：由某个 relation 的某块映射到某个 buffer 槽位。
- 一个共享内存中的哈希表负责维护 tag 到 buffer 槽位的映射。
- `BufferDesc` 负责记录这个 buffer 的元信息和状态；真正的 page 数据放在共享内存的另一个区域（data）里。

关于 buffer pool 的基础实现介绍可以参考PG14 book中的Buffer cache一节。

### 接口与调用栈
★读路径，其调用链大致如下：

```c
ExecSeqScan()
  -> table_scan_getnextslot()
    -> heapgettup() / heapgetpage()
      -> ReadBufferExtended()
        -> ReadBuffer_common()
          -> buffer hash lookup
            -> smgrread() if not found // miss时从磁盘读
```

这个路径在我之前的 [SMGR]({% post_url 2025-09-09-smgr %}) 一文里其实已经从更下层视角提过一次：

```c
Scan(Executor) -> heap(tableAM) -> ReadBuffer(buffer pool) -> smgrread(SMGR) -> FileRead(FD) -> pread(syscall)
```

★写路径，则稍微复杂一点，因为“修改 page”与“把 page 写盘”通常并不是同一个时刻：

```c
ExecInsert() / ExecUpdate()
  -> heap_insert() / heap_update()
    -> ReadBufferExtended(..., RBM_...)
    -> 在 shared buffer 中修改 page
    -> 标脏 MarkBufferDirty() // WAL /checkpoint / bgwriter / backend 后续再处理写盘
```

由上我们可知一个重要的结论：PG中的读写操作一般是只操作buffer pool的，以它中的page作为中转站，并不是直接和磁盘IO打交道（所以你gdb是基本看不到backend进程进行`pwrite()`调用的）。

另外buffer pool 缓存的不只是 heap main fork，也包括 VM/FSM，以及索引页等其他 page。

### Buffer Pool 的核心结构
几个重要结构：
- `BufferTag`：唯一标识一个磁盘块。
- `BufferDesc`：描述一个 buffer 槽位当前状态。
- `BufferStrategyControl`：管理 page replacement 的全局状态。
- `BufferAccessStrategy`：某些访问模式下使用的局部策略，例如 ring buffer。

### BufferTag
`BufferTag` 可以理解成 buffer pool 的 key，核心就是定位一个具体 page：

```c
BufferTag
  rnode           // relfilenode / spcNode / dbNode ...
  forkNum         // main fork, vm, fsm ...
  blockNum        // 第几个block
```

也就是说，只要给定“哪个 relation 的哪个 fork 的哪个块”，就能唯一确定一个 page。

### BufferDesc
如果说 `BufferTag` 是 key，那么 `BufferDesc` 就更像某个 cache entry 的 metadata/header。

可以非常粗略地理解成：

```txt
BufferDesc
  +-- tag             // 当前这个slot缓存的是哪个page
  +-- state           // pin count / dirty / valid / io_in_progress ...
  +-- buf_id          // 对应 shared buffers 里的槽位编号
  +-- content_lock    // 保护page内容
  +-- io_condition... // 与I/O协作相关
```

从逻辑关系上看，大致如下：

```txt
            shared memory

 [BufferDesc 0] ---> [BufferBlock 0 : 8KB page bytes]
 [BufferDesc 1] ---> [BufferBlock 1 : 8KB page bytes]
 [BufferDesc 2] ---> [BufferBlock 2 : 8KB page bytes]
 ...
```

也就是说，`BufferDesc` 和真正的 page 数据通常是分开存放的：
- descriptor 保存“这个槽位现在是什么状态”；
- block 区域保存“这个 page 的实际字节内容”。

`state` 是这里最关键的字段之一，源码里为了性能做了位打包和原子更新，概念上可以先把它理解成下面这些状态的组合：
- refcount / pin count：当前有多少 backend 正在使用这个 buffer。
- usage_count：clock-sweep 替换策略会用到。
- dirty：该 page 是否已被修改但尚未落盘。
- valid：buffer 中的数据是否有效。
- io_in_progress：当前是否正在做 I/O。

### 映射关系与 hash 表
前面说 `BufferTag` 会先查 shared hash table，再定位到某个 `buffer_id`。

这个结构可以简化成：

```txt
(rel,fork,block)
      |
      v
 [BufTable hash]
      |
      +--> buffer 17
      +--> buffer 92
      +--> ...
```

它的作用很直接：防止同一个磁盘 page 被重复装进 shared buffers。

一个典型 miss 流程是：
1. 先拿 `BufferTag` 去查 buffer mapping hash table。
2. 如果 hit，直接拿到已有 buffer。
3. 如果 miss，就要找一个可复用的 victim buffer。
4. 如有必要，先把 victim 的旧脏页刷走。
5. 把新 page 读入这个槽位，并更新 hash table 映射。

因此 buffer hash table 解决的是“怎么找”；而 replacement policy 解决的是“找不到时淘汰谁”。

### 一个简化的读路径
把前面几层合起来，`ReadBuffer_common()` 的典型心智模型大致如下：

```txt
ReadBuffer_common(tag)
    -> 查 hash table
        -> hit: pin existing buffer and return
        -> miss:
             -> 从 freelist / clock-sweep 找 victim
             -> 如有旧映射则删除
             -> 若 victim 是脏页，必要时先写回
             -> smgrread() 读入新page
             -> 建立新 tag -> buffer_id 映射
             -> pin and return
```

真实代码当然远比这个复杂，要处理并发竞争、I/O 协作、扩展 relation、不同 read mode 等情况。

### 并发控制
buffer pool 是数据库里最容易出现高并发争用的地方之一，因为几乎所有数据访问最终都要经过它。

PG 在这里做了多层次并发控制，解决的是不同粒度的问题，分为如下几块：
- hashtable lock：哈希表上的一个多段锁，对 buffer mapping 的并发访问。
- pin buffer：严格来说并不是锁（一致性保护），而是防止 buffer 被替换evict。一般操作顺序是：`pin -> 加锁 -> 操作 -> 解锁 -> unpin`。
- buffer相关的
  - header lock：对 bufferdesc 的并发访问。
  - content lock：对 page 内容的并发控制。
  - io lock：page 上文件IO的并发控制。

仔细思考就能理解为什么需要这么多锁，细节可以参考PG 14 book中Locks->Examples小节以及[这里](https://zhmin.github.io/posts/postgresql-buffer-concurrent/)。

### 脏页与异步写盘
buffer pool 的另一个核心价值，就是把“逻辑修改 page”和“物理写回磁盘”解耦。也就是说，后端修改 page 后，通常只会：
1. 在 shared buffers 里改 page 内容；
2. 写对应 WAL；
3. 把 buffer 标成 dirty；
4. 在之后某个时刻再真正刷盘。

这就是数据库里非常典型的 Write Ahead Log 思路：保证数据的持久性并提升性能。

为什么要这样做？因为如果每次修改一行都同步把 8KB page 写盘，吞吐会非常差。

对 PG 来说，后续写盘大致可能发生在几类场景：
- checkpointer / bgwriter 后台批量刷脏页；（后续会写一篇后台写进程文章来详细讨论）
- backend来写出：某个脏页正好被选中作为 victim，需要先写回；和其他一些情况（见树杰事务一书中的write back章节）。
- checkpoint 要求确保脏页在某个时点前落盘。
- 另外：WAL 与数据页刷盘之间有严格的顺序约束（page上的lsn字段）。

### 页面替换策略
shared buffers 容量总是有限的，因此一点会面临替换问题。缓冲替换算法一般采用LRU，PG也不例外。

不过PG这里的主策略是 clock-sweep，而不是严格的LRU。可以把它想成一个环形指针不断前进：
```txt
 [0] [1] [2] [3] [4] [5]
              ^
           clock hand
```

扫描到某个 buffer 时，大致规则是：
- 如果被 pin 住，跳过。
- 如果 `usage_count > 0`，先减一，再跳过。
- 如果既没被 pin，`usage_count` 也降到 0，就可能成为 victim。

这样做的好处是实现相对简单，而且并不需要维护一个严格排序的全局 LRU 链表，减少了并发热点。

当然，clock-sweep 也不是万能的。比如大范围顺序扫描就很容易把 cache 污染掉，因此 PG 还会配合 access strategy 来缓解。

### BufferAccessStrategy
不是所有访问都应该平等地使用 shared buffers。

例如顺序扫描一个大表时，如果把所有读入页面都当成“热点数据”放进主 buffer pool，往往会把真正高频访问的数据挤掉。

所以 PG 提供了多种 `BufferAccessStrategy`，除了典型的buffer pool访问之外，还有：`bulk read / bulk write / vacuum`。

它们常会配合一个比较小的 ring buffer 使用，让某类大流量顺序访问尽量在局部循环，不去过度污染整个 shared buffers。这也是为什么前面讲 Scan 时会提到 ring buffer 策略。

### 其他

★SLRU

除了 shared buffers 之外，PG 里还有一类非常类似的缓存机制：SLRU（simple LRU）。

它主要服务于另一类特殊的页面数据，例如：`pg_xact`，`pg_multixact`，以及其他事务状态相关页面。

它也位于shared memory 中，并也有页替换和刷盘逻辑，但它和主 buffer pool 不是同一套体系。一个比较粗略的理解是：
- shared buffers：面向普通 relation page，属于通用数据页缓存。
- SLRU：面向一些特定内部目录/状态页的专用缓存。

后者实现更专用，也更朴素。

★pg_buffercache

另外我们可以通过 pg_buffercache 扩展来观察 buffer pool 的行为。它主要提供了一些统计信息，例如：缓存命中率/替换率/命中时间等等。（另外它还是一个我们学习如何写pg扩展的好例子）

```sql
ubuntu=# SELECT * FROM pg_buffercache WHERE relfilenode = (SELECT relfilenode FROM pg_class WHERE relname = 'test2');
 bufferid | relfilenode | reltablespace | reldatabase | relforknumber | relblocknumber | isdirty | usagecount | pinning_backends
----------+-------------+---------------+-------------+---------------+----------------+---------+------------+--------------
      203 |       65539 |          1663 |       16384 |             0 |              0 | f       |          1 |                0
(1 row)
```

一般来说如果大量的share buffer的usagecount都是4或者5，那么这表明你的shared buffer不够大，你应该扩大你的shared buffer；如果大多数的usagecount是0或者1，你就可以尝试减少你的shared buffer，具体减少到多少，可以分析缓存换出命中率。

### Discussion
buffer pool 几乎位于 PostgreSQL 所有数据访问路径的中心位置。很多上层能力，看起来是在优化 scan、vacuum、checkpoint、WAL、table AM，最后都会落回到 buffer pool 的行为上。

理解这部分时比较重要的几个点是：
- 它本质上是 page cache，但又远不只是 page cache。
- 读写路径关注的不是同一件事：读路径更关注命中与定位，写路径更关注脏页与回写。
- 并发控制算法：分层设计，mapping、pin、content lock 各管一层。
- 页面替换和异步写盘，是数据库吞吐能力的重要来源。

如果再往下深挖，这一块就会自然连接到很多后续主题：
- bgwriter / checkpointer
- WAL 与 page flush 顺序
- AIO / prefetch
- table AM / index AM 对 buffer manager 的使用方式

### Questions
1. `pin` 和 `content lock` 分别保护的到底是什么，哪些场景会同时需要二者？
    > 提示：一个防止被替换，一个保护page内容。
2. 当一个脏页被 clock-sweep 选中做 victim 时，具体写回路径是怎样的？
    > 可以顺着 `bufmgr.c` 里的 victim 选择和写回逻辑继续跟。
3. 如何bufferpool中的所有页面被pin住无法被驱逐，那么新页面请求会被阻塞住？
    > 会的，需要分析pin的场景。
4. 前边提到了：除了后台写进程，为什么backend也要回写脏页，这看起来带来了设计/行为上的复杂度（https://mp.weixin.qq.com/s/ifycRL0Rcy7y9AYg2xGtXw）
    > 刷脏和分散sync磁盘热点

(这篇文章AI参与了编辑)