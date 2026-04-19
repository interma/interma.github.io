---
layout: post
title:  "Postgres存储格式"
date:   2025-08-08 10:51:01 +0800
permalink: /postgres/storage-format.html
tags: [postgres, storage]
---

一张数据表中会包含各种类型的字段，其中既有定长数据，也有变长乃至超长数据。这些数据在 PostgreSQL 中是如何存储，并最终返回给用户的呢？

本文简单梳理一下用户数据在 PG 中从磁盘到内存，再到网络传输的大致路径：
```
 磁盘文件        | 共享缓冲区        | 网络 TCP
 heap tuple      → TupleTableSlot → libpq
```

### In Disk
PG 的表数据通常以**无序堆表**的形式存储在磁盘文件中。对比 MySQL InnoDB 这种索引组织表（IOT），PostgreSQL 的主表数据与索引数据是分离存放的。物理文件会继续按段和页切分：
```
main fork   {[page][page]...} {segment files} ...
               │                               
               ▼                               
page        {[header][linepointers]...[tuples]}
                                         │     
                                         │     
tuple       {[header][user-data]} ◄──────┘     
```
★Segment：即数据段文件。关系文件通常根据对象的 `relfilenode` 来命名，并按 1GB 大小切成若干 segment 文件，再追加数字后缀，例如：`17221`, `17221.1`, `17221.2`, ...。
这些段文件在逻辑上共同组成一个更大的连续文件，因此只要给定逻辑偏移量，就能定位到具体的 segment 编号。
> 这里的数据库对象不一定只是表。比如索引和 TOAST 表在 `pg_class` 中也各有自己的条目，因此也有独立的 `relfilenode` 和物理文件，不会和主表数据混存。

★Page：即数据页。每个段文件内部再按 8KB 切分为多个 page。以最常见的 heap page 为例，其内部结构可以概括为：`pageheader | linepointers -> ... <- tuples`。
由于 tuple 是变长的，因此 page 内部需要引入 line pointer 作为定长的寻址结构。这样一来，page 对外就像一个小黑盒，外部只需要通过 lp 下标就能定位其内部的 tuple。
这种结构通常称为 **slotted page**：
```txt
chatgpt画的page内部结构图
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ PageHeader                                                              │
  │  pd_lsn | pd_checksum | pd_flags | pd_lower | pd_upper | pd_special ... │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ ItemId(1)  ItemId(2)  ItemId(3) ...      （行指针数组，向“上”增长）         │
  ├───────────────────────────────自由空间(Free Space)────────────────────── ┤
  │ [HeapTuple N] [HeapTuple N-1] ...        （元组区，向“下”增长）            │
  └─────────────────────────────────────────────────────────────────────────┘
```

lp 号负责在单个 page 内部寻址 tuple；再补充上页号后，就能通过 `(page-no, lp-no)` 唯一定位一个 tuple 条目，因此它通常也被称为 TID（tuple identifier，后面讲索引时还会再提到）。
代码里这两个概念对应的结构体名有些容易混淆：
```c
struct ItemIdData
  // linepointer，指向对应tuple的offset和length
  lp_off;
  lp_len;

struct ItemPointerData
  // tuple-id，唯一标识一个tuple
  ip_blkid;
  ip_posid;
```
> page 内部结构是可定制的，因此不同类型的 page（比如索引页和数据页）会有不同的布局。

在其他一些数据库实现中，也会把全部数据打包到一个巨大文件里。相对来说，我还是更喜欢 PostgreSQL 这种逐层切分的方式。

★Tuple：即元组，对应一行用户数据，内部格式大致可概括为：`tupleheader | userdata[col1, col2, ...]`。它以 tuple header 开头，其中包含 MVCC 相关字段（如 `xmin/xmax`）；随后的 userdata 部分则按照表定义的列顺序保存各列的值。
这些列值大致有几种存储方式：
1. 定长数据，比如 `int`，直接内联存储。
2. 变长数据，比如 `varchar`，采用 **varlena** 格式存储。
3. 超长数据，即 TOAST 数据，会转存到独立的 TOAST 表 `pg_toast_${oid}` 中。

另外，从 PostgreSQL 14 开始，可以为支持 TOAST 的列指定压缩算法：
```sql
ALTER TABLE t1 ALTER COLUMN data SET COMPRESSION lz4; -- pglz or lz4
```

### In Memory
磁盘上的数据会以 page 为单位加载到共享内存中的 shared buffers 中；随后消费方（如执行器）会通过一些内存结构对其进行包装和访问：
```
bufferpool      {[block1][block2]...}◄──────disk pages
                    ▲                                 
tupleslot           │                                 
  - heaptuple ──────┘                                 
  - tupledesc    
---
structure: Datum/HeapTuple/TupleTableSlot
```

★Datum：对列值的抽象，代码里通过它来统一访问一个列值。在 64 位构建中它通常是 8 Bytes，因此对于较长的数据，Datum 往往保存的是一个指针。
> A Datum contains either a value of a pass-by-value type or a pointer to a value of a pass-by-reference type.

```
Datum
  +--> 简单数据（int/float等定长数据，直接按值存储）
  +--> 变长数据（char*/text/blob等，指针引用）
         +--> varlena结构
               +--> varattrib_1b（小变长数据）
               +--> varattrib_4b（大数据，可压缩）
               +--> varattrib_1b_e（TOAST外部存储引用）
```
这里再次提到 varlena（`struct varlena`）：简单说，它就是一个带描述头（如长度信息）的变长数据格式，并且针对短数据做了额外优化。细节见[varlena格式详解](https://zhmin.github.io/posts/postgresql-varlena/)。


★HeapTuple：即堆元组，是对一行数据的内存抽象，基本对应磁盘上的 tuple 格式：
```c
typedef struct HeapTupleData
{
    uint32      t_len;      	// 元组的总大小
    ItemPointerData t_self;     // TID（页号+行号）
    Oid         t_tableOid; 	// 所属表的 OID
    HeapTupleHeader t_data;     // 指向实际的tuple（和磁盘结构基本一致）
} HeapTupleData;
typedef HeapTupleData *HeapTuple;
```
它的 `t_data` 字段指向具体的 tuple 数据，常见有两种用法：
* 指向 shared buffers 中的元组：此时 `t_data` 本质上是一个指针。由于底层页位于共享内存中，因此相关代码必须考虑并发控制。
* 作为单独分配的 tuple 头：`HeapTupleData` 和实际 tuple 数据通常在同一块内存中分配，实际数据紧跟在 `HeapTupleData` 之后。


★TupleTableSlot：执行器中为了便于处理 tuple，会把它与其他必要信息（如元组描述符 tupledesc）一起放进 slot 结构中。这里先点到为止，执行器一节再展开。
> tuple 中的 userdata 本质上是一大块二进制内容，需要结合 tupledesc 才能把各个列值“切”出来。列定义在系统表 `pg_attribute` 中，而取列值的核心逻辑在 `heap_getattr()` 里。

除了 HeapTuple 之外，内存中还有其他几种 tuple 形式：
* MinimalTuple：精简版 HeapTuple，不再保留完整的外围头部信息。执行器做物化时常会把 HeapTuple 转成 Minimal 形式，以节约内存。
* VirtualTuple：更精简，基本只剩 `values + isnull` 标记，多用于执行器算子内部的临时结果。
* MemTuple：PG17 主线代码里已经没有了，GP6 和一些老版本 PG 中可能仍能看到，作用与 VirtualTuple 有些相似。

> 更准确地说，某些 tuple 形式只在执行器中以 slot 的形态出现。比如并没有一个单独的 “VirtualTuple” 结构体，常见的是 `TTSOpsVirtual` 这类 slot 实现。

最后再提一下压缩。需要注意，并不是“整个 heap page 以压缩格式加载到 shared buffers 中”；更常见的情况是 tuple 内部的某些 varlena datum 仍然是压缩态，或者只是一个外部 TOAST 引用。此时它们无法直接按最终值返回给用户，而是会在真正被访问时做**惰性 detoast / 解压缩**，并把结果放到执行器内存（memory context）中。相关逻辑可以顺着 `pg_detoast_datum_XXX()` 一路跟进去看。

### In Network
这一部分严格来说已经不属于“存储”，这里顺便一起带过。

PG 设计了一套构建在 TCP 之上的应用层协议，并通过 libpq 与客户端交互：
1. 先发送 `RowDescription`（可类比 tupledesc），描述结果集字段信息。相关代码可看 `printtup_startup() -> SendRowDescriptionMessage()`。
2. 再逐行发送 `DataRow`。协议支持 text 和 binary 两种返回格式，默认通常是 text。相关代码可看 `printtup()`。
   > 例如 `float` 类型的 `3.14`：text 模式发送字符串 `"3.14"`；binary 模式则发送对应的二进制表示。
3. 客户端收到数据后再自行解析。

如下是RowDescription和DataRow协议包的简化：
```c
// message type 'T'
RowDescription {
    int ncolumns     // 字段数量
    for i in 1..ncolumns {
        string  field_name    // 列名
        int     type_oid      // 类型oid
        int     format        // 0=text,1=binary
        ...
    }
}

// message type 'D'
DataRow {
    int ncolumns  // 字段数量
    for col in 1..ncolumns {
        int   value_len        // 数据长度
        byte[value_len] value  // 实际数据内容
        ...
    }
}
```

libpq协议比较朴实简洁，后续会再单独开一篇文章介绍它。

### Discussion
关于 PostgreSQL 存储的资料非常多，通常也会和事务实现一起介绍，之前提到的几本书里都有比较系统的讲解。
需要特别留意的是：
1. 磁盘上的多版本数据是“带内混合存储”的，这给 PG 带来了一系列独特问题。
2. PG 这套存储是一个典型的行存系统，并和经典执行器结构紧密耦合；对于 AP 业务，如何做列存化改造一直是个大主题。

Questions:
1. null 值是如何存储的？
    > 提示：标记位
2. 数据加载进内存后，压缩态 / TOAST 态的值是什么时候解开的？
    > 提示：前边已经介绍了，另外可以在select*场景下gdb一下何时调用```pg_detoast_datum_packed()```
3. TupleSlot 中的 `tts_values` 数组是干什么的？
    > 提示：看看和 `HeapTuple.t_data` 的关系
