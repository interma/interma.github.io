---
layout: post
title:  "执行器基础"
date:   2025-08-16 09:02:11 +0800
permalink: /postgres/executor.html
tags: [postgres, executor]
---

简要介绍一下 PostgreSQL 中执行器的整体架构（不展开具体算子的实现细节）。

典型算子（如 sort、agg、join）的实现会单独成文；另外，执行器中的一些基础设施（如 memory context、hash table、resource owner 等）也会在后续文章中分别介绍。

### 基础
查询执行器负责执行传入的查询计划（plan tree），并把结果返回给客户端。它位于 query 处理流程的最后一环：
```c
exec_simple_query(const char *query_string)
    //词法和语法分析，return parsetree: xxxStmt -> a raw parsetree (gram.y output)
    parsetree_list = pg_parse_query(query_string); 
    // 语义分析和重写，return querytree: Query
    querytree_list = pg_analyze_and_rewrite_fixedparams(parsetree,...);
    // 查询计划，return plantree: PlannedStmt
    plantree_list = pg_plan_queries(querytree_list,...);
    // 进入portal执行plantree    
    PortalDefineQuery(portal,...,plantree_list,...);
    (void) PortalRun(portal,...,receiver,...);
```
> GP 会把这个过程拆成两部分：生成 plan tree 以及之前的阶段由 QD 完成，随后再把生成的 plan 发给 QE 做 MPP 执行。

这部分 query 处理流程在 PG-JP Book 中有较详细的介绍，另外也可以参考[这里](https://mp.weixin.qq.com/s/YQsQg5H063GzpMDjJJQIaQ)（图示比较丰富）。

### 执行器框架
`ExecutorStart/Run/Finish/End()` 常被视为执行器的几个主要阶段，其中最核心的是 Start、Run 和 End：
```c
PortalRun()
    -> ProcessQuery(PlannedStmt *plan,...)
       {
            QueryDesc *queryDesc = CreateQueryDesc(plan, sourceText, ...);
            ExecutorStart(queryDesc, 0);
            ExecutorRun(queryDesc, ForwardScanDirection, 0, true);
                // -> standard_ExecutorRun() -> ExecutePlan() -> ExecProcNode() -> 具体的算子逻辑比如ExecSeqScan()
            ExecutorFinish(queryDesc);
            ExecutorEnd(queryDesc);
       }
```

以 Seq Scan 为例，大致展开如下：
```c
ExecutorStart(querydesc)
    initPlan()
        ExecInitNode(plan)
            ExecInitSeqScan() // init SeqScanState

ExecutorRun(querydesc)
    ExecutePlan(estate, planstate)
        ExecScan(seqscanstate)
            SeqNext()
                if (scandesc == null)   // desc is a field of state
                    table_beginscan();
                table_scan_getnext();

ExecutorEnd(querydesc)
    ExecEndPlan(planstate)
    // other cleanups
```

下面分别看几个关键阶段。

★`ExecutorStart()`：初始化全局 `EState`，并构造与 plan tree 对应的 planstate tree。

`QueryDesc` 作为根结构，与其他核心结构的关系大致如下：
```
queryDesc
    plannedstmt         // planner's output
    operation           // query type, e.g. select/insert
    dest                // 输出目标 DestReceiver
    ...
    estate              // 执行器全局状态
        es_snapshot         // 快照
        es_range_table      // range table list
        ...
    planstate           // planstate tree，与plantree的结构一一对应
                        // 其中的每个节点都是PlanState结构，保存了节点的内部状态

        ->PlanState
            type            // 多态类型，如SeqScanState->ScanState->PlanState
            plan            // 对应的plan节点
            state           // 全局estate
            lefttree        // 左右子树
            righttree
            ps_ResultTupleDesc  // 结果desc
            ps_ResultTupleSlot  // 结果slot
        ...
```

★`ExecutorEnd()`：沿着 planstate tree 做清理工作。

在它之前还有一个 `ExecutorFinish()`：主要用于处理 query 生命周期中的一些收尾逻辑，也是计时统计里的一个 corner case。
> This routine must be called after the last ExecutorRun call. It performs cleanup such as firing AFTER triggers. It is separate from ExecutorEnd because EXPLAIN ANALYZE needs to include these actions in the total runtime.

★`ExecutorRun()`：执行器的核心逻辑。PG 采用传统火山模型，由父节点持续调用 `next()`，驱动 planstate tree 做**深度优先**遍历。
```
   Output
     ^
     | next()
  [Join]
  ^    ^
  |    | next()
next() |
[Scan][Scan]
```

对 PG 来说，整体流程大致如下，也就是在 planstate tree 上做深度优先遍历：
1. 对 planstate tree 的根节点调用 `ExecProcNode()`。
    ```c
    TupleTableSlot *ExecProcNode(PlanState *node); // 相当于火山模型的next()
    ```
2. 根据节点类型执行对应的特化逻辑，并继续对子节点调用 `ExecProcNode()`。
    > 即控制流是通过函数调用来实现的
3. 如此递归下去，直到叶子节点（通常是某种 scan）为止。
4. 获取到可用 tuple 后，把它填入 `TupleTableSlot`，作为节点间传递数据的“接力棒”，再返回给上层。
   > 换句话说，每个节点都是有状态的，状态就记录在自己的 `PlanState` 中。
5. 最终在根节点处，把结果交给 `DestReceiver` 处理，比如发送给客户端。
    ```c
    void ExecutePlan(QueryDesc *queryDesc, CmdType operation, bool sendTuples, uint64 numberTuples, ScanDirection direction, 
        DestReceiver *dest);
    ```

最后给一段 Run 阶段的典型栈轨迹：
```gdb
#0  ExecSeqScan (pstate=0xc8de28fdb498) at nodeSeqscan.c:109
#2  ExecProcNode (node=node@entry=0xc8de28fdb498) at ../../../src/include/executor/executor.h:274
#3  ExecutePlan (queryDesc=queryDesc@entry=0xc8de28f7f8b0, operation=operation@entry=CMD_SELECT, sendTuples=sendTuples@entry=true, numberTuples=numberTuples@entry=0, direction=direction@entry=ForwardScanDirection, dest=dest@entry=0xc8de28fe4460) at execMain.c:1649
#4  standard_ExecutorRun (queryDesc=0xc8de28f7f8b0, direction=ForwardScanDirection, count=0, execute_once=<optimized out>) at execMain.c:361
#5  ExecutorRun (queryDesc=queryDesc@entry=0xc8de28f7f8b0, direction=direction@entry=ForwardScanDirection, count=count@entry=0, execute_once=execute_once@entry=false) at execMain.c:307
#7  PortalRun (portal=portal@entry=0xc8de28f4f1d0, count=count@entry=9223372036854775807, isTopLevel=isTopLevel@entry=true, run_once=run_once@entry=true, dest=dest@entry=0xc8de28fe4460, altdest=altdest@entry=0xc8de28fe4460, qc=qc@entry=0xffffc2605a68) at pquery.c:766
#8  exec_simple_query (query_string=query_string@entry=0xc8de28ecf690 "select * from test;") at postgres.c:1278
```

### TupleTableSlot
`TupleTableSlot` 是执行器里非常关键的基础设施。它统一了多种 tuple 表示形式，并尽量避免过早物化。

可以把它理解成“执行器内部统一的行容器”，核心点有两个：
- 适配多种 tuple 形态：既可以包装 buffer 中的物理 tuple，也可以包装 heap tuple、minimal tuple，或者只保存 `Datum/isnull` 数组的 virtual tuple。
- 尽量惰性展开：很多时候 slot 里并不会立刻把所有列都解析进 `tts_values`，而是在真正访问某列时才做 deform，以减少不必要的 CPU 开销。
```
 * 1. physical tuple in a disk buffer page (TTSOpsBufferHeapTuple)
 * 2. physical tuple constructed in palloc'ed memory (TTSOpsHeapTuple)
 * 3. "minimal" physical tuple constructed in palloc'ed memory (TTSOpsMinimalTuple)
 * 4. "virtual" tuple consisting of Datum/isnull arrays (TTSOpsVirtual)
```

### Discussion
对于执行器来说，首要目标是“准确”，其次才是“高效”。真正的挑战主要在如何更高效地执行，尤其是在 OLAP 场景下。

PG 目前的火山模型执行器诞生于磁盘 I/O 是主要瓶颈的年代，同时当时 CPU 的并发能力也远不如今天。放到现代硬件上看，它已经逐步成为重要的性能瓶颈之一。

目前主要优化手段包括：
- 宏观架构：更好的并行。
  - 并行执行：如 PG 已有的 parallel worker（算子内并行）、GP 的多节点 MPP 执行等。
  - pipeline 执行器：以批量数据为处理对象，数据流由下向上，以 push-based 方式执行。
- 指令优化：尽量减少 CPU cycle。
  - 向量化执行：算子之间传递 batch，而不是一次一个 tuple，以减少函数调用开销。
  - SIMD/JIT：使用或生成更高效的 CPU 指令。PG 已经在表达式计算上使用 LLVM 做 JIT 优化。
- 通用算法与微优化。
  - 例如延迟物化、多阶段 agg、share input scan、runtime filter 等。

在具体工程实现上，需要组合使用如上的多种优化手段，并在存储层上搭配合适的列存实现（从而更好的节约磁盘IO）。

对 PG 而言，受限于工程复杂度，目前更倾向于复用已有基础设施（如 LLVM），并优先选择对整体架构侵入较小的方向来优化。这只是我的个人观察。相比之下，其他开源数据系统里，以 Impala、ClickHouse 为代表，已经在这方面做了大量工作。可以参考的一些项目包括：
- DuckDB：实现了非常简洁的 push-based pipeline 执行器。
- `pg_duckdb`：把 DuckDB 的执行能力以更直接的方式带到 PG 生态中。
- Doris：整合了许多已知的执行优化手段，可作为列存计算系统的一个综合参考。
- [hydra-columnar](https://github.com/hydradatabase/columnar)：PG 的一个列存扩展实现，其中执行器部分实现了 query 并行和向量化，适合作为学习入口。

论文方面可以重点关注慕尼黑大学（TUM）和阿姆斯特丹 CWI 的一系列相关工作。
