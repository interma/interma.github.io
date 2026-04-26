---
layout: post
title:  "对 PG 进行扩展"
date:   2026-04-26 15:19:23 +0800
permalink: /postgres/extension.html
tags: [postgres, extension]
---

PG 的一个很大魅力，就是它并不只是一个“固定功能”的数据库内核，而是预留了相当多可扩展入口。

轻量一些的需求，可以只写 SQL 函数或 C UDF；再重一些，可以通过 hook 改写内核行为；如果还不够，就继续往执行器、访问方法、外部数据访问甚至后台任务框架里深入。

不过，“PG 扩展”这个词本身有两层含义：
- 一层是 packaging 意义上的 extension，即 `CREATE EXTENSION xxx` 这种可安装模块。
- 另一层是更广义的 extensibility，即 PostgreSQL 内核提供了哪些能力边界，允许你把新逻辑插进去。

本文章侧重于介绍第二个方面。

### 基础
一个粗略的可扩展点的分层图：

```txt
SQL 层
  ├─ function / aggregate / operator / type
规划与执行
  ├─ planner hook
  ├─ executor hook
  └─ Custom Path / Custom Scan
存储访问层
  ├─ index AM
  └─ table AM
对外连接与后台任务
  ├─ FDW
  └─ background worker
```

> 留意：在Sql词法/语法层面是没有hook的。所以要新增语法特性的话，只能通过函数、类型、operator 来实现。或者直接硬编码到pg代码中。

如果按“侵入性”和“实现难度”来排，大致也是从上到下逐步增加：
- SQL / C 函数：最容易上手，适合新增计算逻辑。
- hook：适合在已有流程上做全局增强或观测。
- Custom Scan / Custom Path：适合把某类查询改写成自定义执行逻辑。
- index AM / table AM：适合深度改造存储与访问方式，复杂度很高。
- FDW / background worker：适合连接外部系统，或在数据库内长期运行后台任务。

对大多数业务需求来说，其实很少一上来就需要碰 AM 或执行器节点。通常是先从“能不能用函数、类型、operator、hook 解决”开始。

### SQL / C 函数与自定义类型
这是最常见、也最“正统”的扩展入口。

★函数

最简单的情况，是新增一个 SQL 函数，甚至只是对已有函数做组合：

```sql
CREATE FUNCTION my_add(a int, b int)
RETURNS int
LANGUAGE SQL
AS $$
    SELECT a + b
$$;
```

如果性能敏感，或者需要直接操作 PG 内部数据结构，就可以继续往 C 函数走：

```c
PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(my_add);

Datum
my_add(PG_FUNCTION_ARGS)
{
    int32 a = PG_GETARG_INT32(0);
    int32 b = PG_GETARG_INT32(1);
    PG_RETURN_INT32(a + b);
}
```

这里最核心的点是：PostgreSQL 把函数看成非常基础的可扩展单元。很多看起来像“语法特性”的东西，底层其实都能回到函数机制上，比如：
- 普通标量函数
- aggregate / window function
- operator 背后的实现函数
- 某些 type 的输入输出函数

因此如果你的需求本质上是“新增一种计算”，优先考虑函数往往是最自然的路径。

★自定义类型

再进一步，就是自定义类型。比如：
- 一个新的压缩向量类型
- 特殊编码格式的二进制类型
- 带额外语义的 domain / enum / range

自定义类型本身通常并不是孤立存在的，而是要和下面这些能力配套：
- 输入输出函数：告诉 PG 如何解析和打印这个类型。
- 比较函数与 operator：让它能参与表达式计算。
- opclass / opfamily：让它能真正接入索引体系。
- selectivity / statistics 支持：让优化器对它“有感觉”。

这也是为什么很多看起来只是“加一个类型”的扩展，最后会越做越大。比如 `pgvector` 的核心价值并不只是“新增 vector 类型”，而是围绕这个类型继续补上距离计算、索引支持和查询语义。

如果是类型这一侧，可以先记住这几个关键词：
- `TypeName` / `Oid`：SQL 层类型最终都会映射到 catalog 里的类型 OID。
- 输入输出函数：通常就是 `xxx_in` / `xxx_out` 这类入口。
- `pg_type`：类型元信息最终注册到这里。
- `pg_opclass` / `pg_opfamily`：类型要接入索引体系时经常会碰到。

★打包成 extension

如果要做成可安装模块，通常还会有这样一层目录结构：

```txt
myext/
  ├─ myext.control
  ├─ sql/
  │   └─ myext--1.0.sql
  └─ src/
      └─ myext.c
```

大致过程如下：

```txt
CREATE EXTENSION myext
    -> 读取 myext.control
    -> 执行 sql/myext--1.0.sql
    -> 在需要时加载动态库
    -> 把 SQL 对象注册进系统 catalog
```

### hook 机制
如果说函数更像“新增能力”，那么 hook 更像“在内核既有流程中插一刀”。

典型 hook 包括：
- `planner_hook`：介入查询规划流程，可以做 plan 前后的观测、改写或替换规划策略。
- `ProcessUtility_hook`：介入 DDL / utility 语句处理流程，比如 `CREATE`、`ALTER`、`COPY`、`EXPLAIN` 等。
- `ExecutorStart_hook` / `ExecutorRun_hook` / `ExecutorEnd_hook`：介入执行器生命周期，适合做执行期统计、审计和运行控制。
- `object_access_hook`：在 catalog 对象被创建、修改、删除等操作时触发，适合做对象级审计或限制。
- `emit_log_hook`：介入日志输出路径，适合做日志转发、附加字段或自定义日志处理。

一个典型模式如下：

```c
static planner_hook_type prev_planner_hook = NULL;

static PlannedStmt *
my_planner(Query *parse, const char *query_string,
           int cursor_opts, ParamListInfo bound_params)
{
    /* 在这里做观测、改写或注入额外逻辑 */

    if (prev_planner_hook)
        return prev_planner_hook(parse, query_string, cursor_opts, bound_params);
    else
        return standard_planner(parse, query_string, cursor_opts, bound_params);
}
```

这里要特别注意两点：
- hook 一般要做链式调用，不能粗暴覆盖掉前一个扩展的 hook。
- 很多 hook 并不是稳定 ABI，更像“源码级扩展点”；PG 大版本升级后，扩展代码经常也要跟着适配。

hook相当于给内核行为打补丁式增强，比如 `pg_stat_statements`、`auto_explain` 这类扩展，就大量利用了 hook。

### 自定义执行器节点
如果 hook 还不够，而你的需求已经不是“在已有节点外面包一层逻辑”，而是“我想让优化器和执行器真正认识一种新节点”，那么就会走到 Custom Path / Custom Scan。

这部分是 PostgreSQL 留给扩展实现自定义执行逻辑的一个重要入口。

★为什么需要 Custom Scan

有些场景下，简单函数调用并不能表达你的意图，例如：
- 扫描的数据并不来自普通 heap，而来自外部列存或自定义缓存层
- 某段计算希望整体下推到外部引擎执行
- 需要一个新的执行节点，在运行时维护自己的状态机

这时如果还硬塞进现有节点模型里，往往会比较别扭。Custom Scan 的意义，就是让扩展能在执行器 plan tree 中占一个正式位置。

★大致流程

典型链路是：

```txt
set_rel_pathlist_hook / planner hook
    -> 生成 CustomPath
    -> planner 选择该 path
    -> create_plan 生成 CustomScan plan 节点
    -> ExecutorStart 时初始化 CustomScanState
    -> ExecutorRun 时通过自定义回调产出 tuple
```

也就是说，Custom Scan 通常不只是“执行器节点”这么简单，它往往要同时碰到规划阶段和执行阶段。

一个很粗略的理解方式是：
- `CustomPath`：告诉优化器“我这里有另一条可选路径”。
- `CustomScan`：把这条路径固化进 plan tree。
- `CustomScanState`：执行时保存节点内部状态。

这类扩展通常适合：
- 列存扫描
- 远端 pushdown
- 向量化执行
- 特殊的 join / scan / cache 节点

`pg_mooncake`、`hydra-columnar` 这类项目都值得结合着看。

### 自定义索引 AM / Table AM
再往下，就是访问数据的扩展点了。

PostgreSQL 把表访问和索引访问都抽象成了访问方法（access method）接口。扩展可以自己实现一套 AM，然后挂到 SQL 层暴露出来。

★index AM

索引 AM 的典型例子有：
- btree
- hash
- gist
- gin
- brin

如果只是想先理解 PG 内核里“索引访问方法”大概长什么样，可以先参考之前的[《Btree索引》]({% post_url 2025-09-28-btree %})。

这些在内核里都是不同的索引访问方法。理论上，扩展也可以新增自己的索引 AM。

这类扩展通常需要回答一整套问题：
- 索引页如何组织？
- 如何插入、删除、扫描？
- 如何处理 WAL、recovery、并发控制？
- 如何支持 vacuum？
- 如何向优化器暴露代价估算与能力边界？

所以它并不是“写几个回调函数”那么简单，而是几乎等于做一个小型存储引擎组件。

★table AM

table AM 更进一步，它决定了“表”本身怎么存、怎么扫、怎么更新。heap AM 只是其中一种实现。

如果你想做下面这些事情，就有可能碰到 table AM：
- 行存改列存
- 特殊压缩格式
- 面向冷数据的归档存储
- 特化的 MVCC / vacuum 行为

但现实里，真正做 table AM 扩展的门槛非常高，因为它要和 PG 的大量基础设施配合：
- buffer manager
- WAL / recovery
- visibility / snapshot
- vacuum / analyze
- executor 中各类 scan/update 路径

### FDW 与 background worker
前面几类更偏“扩展内核已有数据路径”；而 FDW 和 background worker，则更偏“接入外部系统”与“在数据库里跑长期任务”。

★FDW
FDW（Foreign Data Wrapper）让 PG 可以把外部数据源映射成 foreign table，从 SQL 层看起来几乎像普通表：

```sql
CREATE FOREIGN TABLE ext_t (
    id bigint,
    name text
)
SERVER my_server
OPTIONS (...);
```

常见用途包括：
- 访问远端 PostgreSQL、MySQL、对象存储或其他数据系统
- 做 federated query
- 把部分过滤、排序、聚合下推到远端

FDW 的优点是接口边界相对清晰，和 SQL 语义也比较贴近；很多“从外部读数据”的需求，优先考虑 FDW 往往比直接魔改执行器更稳妥。

★background worker
background worker 则允许你在 PG 内部启动长期运行的后台进程。

常见用途：
- 异步任务调度
- cache 预热
- 定时维护
- 与外部系统做流式同步

它通常和下面这些机制一起出现：
- shared memory
- latch / signal
- shmem startup hook
- libpq / SPI

如果需求是“不是由用户 SQL 直接触发，而是需要数据库自己在后台持续做事”，那 background worker 往往就是最直接的答案。

### 如何选择扩展入口
从学习顺序上，建议按这个顺序往下走：

1. 只是新增计算逻辑？
    - 先看 SQL 函数 / C UDF / aggregate / operator。
2. 想增强某个全局阶段的行为？
    - 看 hook。
3. 想让优化器或执行器识别一类新节点？
    - 看 Custom Path / Custom Scan。
4. 想对接外部数据系统？
    - 优先看 FDW。
5. 想在数据库里常驻运行后台任务？
    - 看 background worker。
6. 想重做存储组织或索引结构？
    - 才考虑 table AM / index AM。

### Discussion
对 PG 来说，扩展能力是它生态繁荣的一个根本原因。

很多成功项目，本质上都可以看作是“找对了扩展入口”：
- `pg_stat_statements`：主要是观测与统计增强。
- `pg_trgm`：类型 / operator / index 配套扩展。
- `pgvector`：类型、函数、operator、索引能力的组合。
- `postgres_fdw`：把远端 PG 接进本地优化器和执行器。
- `timescaledb`：在 hook、planner、catalog、后台任务等多处联合扩展。

不过也要看到，PG 的很多扩展点更接近“源码接口”，而不是稳定 ABI。扩展做得越深，越容易在大版本升级时付出适配成本。

所以对扩展开发来说，真正重要的不只是“能不能做”，更是：
- 这个入口是否足够稳定？
- 是否和优化器/执行器/存储层现有模型契合？
- 是否值得为了这个需求承担后续维护成本？

### Questions
1. 如果要给 PG 增加一个“列存扫描”能力，最佳入口到底是 FDW、Custom Scan，还是 table AM？
    > 大概率要根据是否需要 MVCC 深度融合、是否要对优化器暴露原生能力来判断。
2. hook 和 Custom Scan 的边界在哪里？
    > 一个常见判断方式是：如果只是“影响已有流程”，更偏 hook；如果要“引入新的 plan/executor 节点语义”，更偏 Custom Scan。
3. 如果只是想加一个新索引，是否一定要自定义 index AM？
    > 不一定。很多时候真正需要的是 opclass/operator 支持，而不是整套新的 AM。

(这篇文章AI参与了编辑)
