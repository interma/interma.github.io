---
layout: post
title:  "更好的列存实现"
date:   2026-06-02 15:23:00 +0800
permalink: /greenplum/better-columnar.html
tags: [greenplum, columnar]
---

GP 作为一个 OLAP 数据库，提供了自己的列存实现：[AOCS 表](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-ddl-ddl-storage.html#topic39)。

但一个长期让我困惑的现象是：在不少性能测试里，AOCS 表的收益看起来并没有想象中那么明显；而很多客户选择 AOCS，更多也只是为了节省磁盘空间，而不是追求显著的查询加速。

后来随着见识增长，才逐渐意识到：如果只是把 row store 换成 column store，并不会自动得到一个更适合 OLAP 的数据库。真正有价值的列存实现，还需要在查询计划和执行层面一起配套优化。

### 基础
一个“更好的列存实现”至少要同时回答下面几个问题：

1. 数据怎么排布？
    - 是纯列式、PAX，还是 row group + column chunk？
2. 如何减少 I/O？
    - 只读需要的列，并尽量利用压缩、late materialization 等技术。
3. 如何吃满 CPU？
    - 不能只靠单线程顺序解压和逐 tuple 解释执行，而要配合并行和向量化。
4. 如何处理写入与可见性？
    - OLAP 虽然读多写少，但 update / delete 以及后续维护仍然绕不过去。
5. 如何接入优化器和执行器？
    - 不然存储层再好，也很难稳定选出正确计划。

也就是说，列式存储、执行优化和查询优化其实是强耦合的三件事。像 GP 的 AOCS 这样，如果只把“存储格式”这一项单独做出来，收益通常都不会特别彻底。

### 存储层
对列存来说，最基本的目标当然是“只读需要的列”。但真正的工程实现一般不会把整张表简单拆成 N 个独立列文件，而更常见的是 row group + column chunk 的组织方式：

```txt
[Row Group 1]          [Row Group 2]
  ├─ col_a chunk         ├─ col_a chunk
  ├─ col_b chunk         ├─ col_b chunk
  ├─ col_c chunk         ├─ col_c chunk
  └─ metadata            └─ metadata
```

这样做有几个好处：
- row group 是天然的调度粒度，适合并行切分。
- 每列单独编码压缩，压缩率通常更高。
- 元信息也能按 row group 维护，比如 min/max、null count、行数等。
- 查询只读部分列时，不需要把整行从磁盘拖出来。

由此可知，列存的收益并不只是“磁盘读得少”，还包括“CPU 上少做很多无效工作”。如果查询只关心 `sum(price)` 和 `where shipdate >= ...`，那就不应该把几十个无关列也一起解出来。

很多系统在第一版列存实现里，往往已经做到了基础的磁盘列式存储；但如果执行器仍然需要先把它们转换成内存里的行式 tuple，再对接传统执行器去跑，那么 CPU 端很快又会成为新的瓶颈。

### 执行器
在现代硬件上，常见瓶颈往往已经不只是磁盘 I/O，而是很容易进一步转移到 CPU 上，比如函数调用开销、分支预测失败、cache miss 等等。所以，一个更好的列存实现，必须把“如何更高效地算”也纳入设计目标。

PG / GP 传统的 Volcano 执行器诞生于“磁盘 I/O 是主要瓶颈”的年代，因此对 CPU 开销的打磨相对较少。

而更现代的执行器，往往会引入如下优化：
* 向量化执行（per batch）：减少函数调用开销，降低缓存压力。
* SIMD 指令集优化：单位 CPU cycle 内完成更多的任务。
* 并行执行（算子内，算子间）：充分利用多核并行计算能力。
* push-based pipeline 执行（DAG）：减少数据复制，提高数据传输效率。
* 针对特定算子进行改造：比如 filter / agg 等算子，可以通过向量化执行进一步提高效率。

如下就是一个非常简化的向量化 `filter / agg` 的流程（pushed pipeline）：
```txt
column chunks
    -> decode to vectors:   shipdate[]   discount[]   price[]
    -> filter in batch
    -> selection vector:    [1,0,1,1,0,...]
    -> gather price[]
    -> vectorized agg
    -> partial sum
```

另外，对执行器来说，也需要尽量避免“过早退化回 tuple”。一个比较理想的方式是：
- scan 先输出 batch
- filter / project / aggregate 尽量继续吃 batch
- 只有在确实需要和 row-based 节点对接时，才做 tuple materialization

### 优化器
列存不是“底下悄悄换一种文件格式”就完事了，优化器也必须知道它的存在。

比如，可以把下面几类信息暴露给优化器：
- metadata 中的各种信息：row group 数量、大小、压缩率、min/max、null count、行数等
- zone map 可跳过的数据比例
- 是否支持更高效的聚合、过滤、projection
- 并行度策略

如果这些信息没有进入代价模型，那么优化器就很难稳定地选出真正适合列存的执行路径。

### 写入与删除
尽管基于列存的 OLAP 系统主要优化的是读操作，但在工程上，一个绕不开的问题仍然是如何实现写操作（insert / delete / update）。

一个常见思路是基于 `delete bitmap` 做配套设计，典型实现通常会带有下面这些特征：
- append-friendly：批量写入新数据
- delete bitmap：记录被删除的逻辑行
- delta store：把小批量 update / insert 先落到更适合写的区域
- compaction：后台合并小文件、清理 delete bitmap、重写编码

关于列存写操作的实现细节讨论，可以看[这里](https://zhuanlan.zhihu.com/p/665260177)。这部分最核心的问题之一，是确定“更新的原子单位”到底是什么。

### Discussion
多年前，如何设计并实现一个高性能列存系统，还是一个非常复杂的话题。尽管大多数方向上已经有比较明确的学术论文指引，但工程复杂度依然很高，真正成熟的开源实现并不多。

不过随着时代发展，ClickHouse、Doris、DuckDB 等项目逐渐成熟，列存系统已经不再是少数人掌握的“黑魔法”了。
* 列存算是 OLAP 数仓系统的一个子集，相关资料可以看这个[论文集](https://zhuanlan.zhihu.com/p/2020991787376878585)。
* DuckDB 算是这类系统里非常好的学习对象，它的[系列介绍](https://mp.weixin.qq.com/s/s7XkUrPjs_NAgVsun2zZGg)质量很高，而且代码量也不算特别大。

最后回到 GP，我个人感觉：单机侧的列存优化，例如向量化、算子内并行等，其实更适合尽量复用 PG 社区已有的积累，避免重复造轮子；而 GP 自己更值得投入的，还是分布式执行器和优化器这一层的改造。

毕竟对 GP 来说，真正拉开差距的地方往往不在“单个节点能不能把一批列数据算得更快”，而在于多节点之间如何切分任务、如何做 motion / interconnect、如何控制数据倾斜，以及优化器能否稳定选出一个足够好的分布式计划。再结合GP的成熟稳定和高兼容PG，这才GP的核心竞争力。

(这篇文章AI也参与了编辑)
