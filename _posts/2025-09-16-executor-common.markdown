---
layout: post
title:  "执行器基础"
date:   2025-08-16 09:02:11 +0800
permalink: /postgres/executor.html
tags: [postgres, executor]
---
简要介绍一下postgres中执行器的整体架构（并不包括具体的算子实现）。

对典型算子（如sort，agg，join等）的实现介绍会单独起一篇文章，另外执行器中的主要基础设施（如memory context，hash table，resource owner等）也会后续单独起一篇来介绍。

### 基础
查询执行器负责执行传入的查询计划（plantree），然后返回结果给用户端，其位于query流程的最后一步：
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
> GP将这个过程切分为2块：plantree以及它之前的部分放到QD上，然后再将生成的plan发送给QE来进行MPP执行

这部分query处理流程在PG-JP Book中有很详细的介绍，另外也可以参考[这里](https://mp.weixin.qq.com/s/YQsQg5H063GzpMDjJJQIaQ)（有丰富的图示）

### 执行器框架
Executor-Start/Run/End()俗称执行器三部曲，它们组成了执行器的3个主要阶段：
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

★```ExecutorStart()```初始化了全局EState结构和planstate tree（和plantree对应）

queryDesc作为根结构，它和其他主要结构之间的层次关系如下：
```
queryDesc
    plannedstmt	        // planner's output
    operation           // query type, e.g. select/insert
    dest                // 输出目标DestReceiver
    ...
    planTree            // plan tree
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

★```ExecutorEnd()```沿着planstate tree进行清理工作

在它之前还有一个```ExecutorFinish()```逻辑：用于处理query计时的一个corner case
> This routine must be called after the last ExecutorRun call. It performs cleanup such as firing AFTER triggers. It is separate from ExecutorEnd because EXPLAIN ANALYZE needs to include these actions in the total runtime.

★```ExecutorRun()```是执行器的核心逻辑，使用了传统火山模型：通过父节点连续调用```next()```驱动在PlanTree上进行**深度优先**遍历
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

对于pg来说，具体的流程如下（在PlanStateTree上进行深度优先遍历）：
1. 对PlanState树的root节点调用```ExecProcNode()```
    ```c
    TupleTableSlot *ExecProcNode(PlanState *node); // 相当于火山模型的next()
    ```
1. 根据节点的类型不同，执行各自的特化逻辑，并在它的子节点上继续调用```ExecProcNode()```
    > 即控制流是通过函数调用来实现的
1. 深度优先递归调用，直到叶子节点（往往是一个scan）为止
1. 获取到可用的tuple后，将它填入`TupleTableSlot`（作为节点间数据交互的"接力棒"），并返回给上层使用
    > 即每个节点都是有状态的，状态信息就记录在当前的PlanState中
1. 最终在root节点处，将结果交给```DestReceiver```来处理（比如传输给客户端）
    ```c
    void ExecutePlan(QueryDesc *queryDesc, CmdType operation, bool sendTuples, uint64 numberTuples, ScanDirection direction, 
        DestReceiver *dest);
    ```

下边是以hashjoin为例的框图：
```gdb
ChatGPT画的递归调用图

                 +--------------------+
                 | 1.HashJoinState    |   <-- PlanState 节点
                 |  ExecProcNode()    |
                 +--------------------+
                       |        |
            Build side |        | Probe side
           (右子树, inner)       (左子树, outer)
                       |        |
        ----------------        ----------------
        |                                     |
        v                                     v
+---------------------+            +----------------------+
|  2.Hash node        |            |   4.SeqScan (outer)  |
|   ExecProcNode()    |            |    ExecProcNode()    |
+---------------------+            +----------------------+
          |                                   |
   produce inner tuples                 produce outer tuples
          |                                   |
          |                                   |
          |--- 3.build hash table ----------->|
                                              |
             probe with outer tuple ----------|
                                              v
                                5.存放到TupleTableSlot，并返回给上层
```

以及栈轨迹中的关键函数：
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
(todo)
```
 * 1. physical tuple in a disk buffer page (TTSOpsBufferHeapTuple)
 * 2. physical tuple constructed in palloc'ed memory (TTSOpsHeapTuple)
 * 3. "minimal" physical tuple constructed in palloc'ed memory (TTSOpsMinimalTuple)
 * 4. "virtual" tuple consisting of Datum/isnull arrays (TTSOpsVirtual)
```

### Discussion
对于执行器来说，需要做到的是"准确高效"。准确是最基本要求，因此对它的挑战主要在如何高效执行上（特别是对于olap场景）。

而目前pg的火山模型执行器来自于**远古时代**（当时磁盘IO是主要开销，同时cpu并发能力也不强）。而对于现代计算机系统架构，它已经逐步成为了性能瓶颈。

目前采用的主要优化手段包括：
- 宏观架构 → 更好的并行
    - 并行执行：如pg已经实现了的parallel worker（算子内），gp的多节点mpp执行等
    - push-based pipeline执行器：以批量数据为操作对象，让数据流以从下至上（而不是火山模型的由上至下）方式来执行
- 指令集优化 → 更精简的cpu-cycle
    - 向量化执行：算子之间采用batch结构，传输一批数据（而不是一个tuple），减少函数调用开销
    - SIMD/JIT优化：生成更高效的cpu指令。pg已经针对表达式计算引入了llvm进行jit优化
- 通用算法
    - 一些通用的微优化，如：延迟物化，多阶段agg，shareinputscan，runtime filter等等

具体实现时，往往需要组合使用如上的多种优化手段，并在存储层上搭配合适的列存实现（更好的节约IO）。

可以参考的一些开源项目：
- duckdb，实现了push-based pipeline执行器
    - pg_duckdb，更直接的将执行器直接搬到了pg heap行存上
- doris，整合了如上多种优化手段，（据说）是集大成者
- [hydra-columnar](https://github.com/hydradatabase/columnar)：pg的一个列存实现（entension方式），其中的执行器部分实现了query并行和向量化
