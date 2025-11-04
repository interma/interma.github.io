---
layout: post
title:  "Postgres存储格式"
date:   2025-08-08 10:51:01 +0800
permalink: /postgres/storage-format.html
tags: [postgres, storage]
---

一张数据表中有各种类型的字段，其中存储了定长，变长乃至超长的数据。这些数据是如何存储在postgres中并返回给用户使用的呢？

这里简单介绍一下用户数据在pg中从磁盘到内存的存储格式，乃至到网络的传输方式：
```
 磁盘 file    | 内存 share buffer | 网络 TCP  
  HeapTuple  →    TupleSlot     →   libpq  
```

### In Disk
pg的数据以**无序**堆表（对比mysql innodb，它是索引组织表IOT）的形式存储在磁盘上的文件中，这些文件按段/页切分：
```
main fork   {[page][page]...} {segment files} ...
               │                               
               ▼                               
page        {[header][linepointers]...[tuples]}
                                         │     
                                         │     
tuple       {[header][user-data]} ◄──────┘     
```
★Segment：即数据段文件，磁盘上的数据文件是根据对象的```pg_class.relfilenode```来命名。按1GB大小切成了若干segment段文件，并标记上数字后缀，比如：```17221, 17221.1, 17221.2，...```。
这些段文件在逻辑上构成了一个无限大小的文件，根据偏移量就能定位到具体的段文件编号了。
>留意这里数据库对象不一定是表，比如index索引和toast表在pg_class中都有自己的条目，因此也有独立的relfilenode和对应的物理文件，它们不和主表数据混合存储在一起的

★Page：即数据页，每个段文件内部以8KB大小再细切为多个page。以最常见的heap page为例，其内部结构为：```pageheader|linepointers->...<-tuples```。tuple代表了一行用户数据，由于tuple是变长的数据，因此引入了linepointer行指针（作为一个定长转变长的跳转结构）。于是将page变成了一个小黑盒，外部通过lp的下标来一一对应其内部的tuple。
这种结构的学名似乎叫**slotted page**：
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

lp号对一个page内部的tuple进行了寻址，再补充上页号后，就能通过```(page-no, lp-no)```来唯一定位一个具体tuple条目，因此它也通常被叫做TID（tuple-id，索引中会再提到）。
留意代码中它们2个的结构体名有点容易混淆：
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
> page内部结构都是可以自定义的，不同类型的page（比如索引页和数据页）的结构显然是不同的

在其他一些数据库实现中，也有将全部数据打包存储到一个巨大文件中。相对来说，我还是比较喜欢postgres这种逐层切分的方式。

★Tuple：即元组，它对应了一行用户数据，内部格式：```tupleheader|userdata[col1,col2,...]```。以tupleheader（mvcc相关字段就在里边，如xmin/xmax）开头，随后的userdata部分中保存这行数据的多个列值（按表的列定义顺序）。
这些列值的存储方式分为下几种情况：
1. 定长数据，比如int，直接存储
1. 变长数据，比如varchar，采用**varlena格式**（下节再进行介绍）存储
1. 超长数据，即toast数据，另起一个toast表```pg_toast_${oid}```来存储

另外从postgres14开始是支持列压缩的：
```sql
ALTER TABLE t1 ALTER COLUMN data SET COMPRESSION lz4; -- pglz or lz4
```

### In Memory
磁盘上的数据会以page为单位放到位于共享内存的bufferpool中，然后消费方（如执行器）会通过一些内存结构对它们进行包装使用：
```
bufferpool      {[block1][block2]...}◄──────disk pages
                    ▲                                 
tupleslot           │                                 
  - heaptuple ──────┘                                 
  - tupledesc    
---
structure: Datum/HeapTuple/TupleTableSlot
```

★Datum：对列值的抽象，通过它来访问一个列值。一般大小为8Bytes，因此对于长数据，实际上它是一个指针。
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
这里再一次提到了varlena结构（```struct varlena```）：简单说它就是一个带描述头（如长度信息）的变长数据格式，另外针对短数据进行了一些优化，细节见[varlena格式详解](https://zhmin.github.io/posts/postgresql-varlena/)


★HeapTuple：即堆元组，对一行数据的抽象，基本是磁盘上tuple格式的直接对应：
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
它的t_data字段保存了具体的tuple数据，这里主要有二类用法：
* 指向bufferpool中的元组: 在这种情况下，t_data是一个指针。既然bufferpool是共享的，相关代码一定要考虑并发控制
* 作为分配的tuple头: HeapTupleData和实际的tuple数据通常在同一块内存中分配，实际数据紧跟在 HeapTupleData结构的内存之后


★TupleTableSlot：执行器中为了方便使用tuple，将它进行初步解析并搭配其他必要数据（如元组描述符tupledesc）一起放到这个slot结构中。这里先简单提一下，执行器一节会再详细介绍。
> 关于tupledesc：tuple中的userdata是一大块二进制内容，需要配合tupledesc将各个列值"切"出来。列定义在系统表```pg_attribute```中，切列值的逻辑在```heap_getattr()```中

除了HeapTuple之外，内存中还有其他几种tuple类型：
* MinimalTuple，精简版heaptuple，不再包含header信息了。执行器在做物化的时候往往将HeapTuple转化为Minimal形式，以节约内存
* VirtualTuple，更精简的tuple格式，基本只剩values+null标记了，多用在执行器算子内作为临时存储
* MemTuple，PG17代码中已经没有了，GP6和老版本的PG（可能）还有，功能和VirtualTuple有点类似

> 更准确的说，某些tuple类型只在执行器中使用，因此只有slot结构：比如是没有VirtualTuple的，只有VirtualTupleSlot结构

最后，再提一下压缩：直接加载在bufferpool中的页数据可能是压缩形式，那么heaptuple.t_data所指向的这个压缩数据是无法直接返回给用户的。会根据具体需要，择时进行**惰性解压缩**并转储到执行器内存（memorycontext）中，这个逻辑在```pg_detoast_datum_XXX()```中。

### In Network
实际上不算是存储了，这里一起简单介绍一下。

pg设计了构建于TCP之上的应用层libpq协议，通过它传输数据给客户端：
1. 首先发送RowDescription信息（对应tupledesc），它描述了结果的字段情况，细节见函数`printtup_startup() → SendRowDescriptionMessage()`
2. 然后逐行发送DataRow（对应tuple），libpq协议中有2种可选返回格式：text和binary（`pgresAttDesc.format`），默认是text型。细节见函数`printtup()`
  > 比如float类型3.14：text模式下发送字符串"3.14"，binary模式下发送IEEE754的4bytes浮点数
3. 客户端收到数据后再自行解析

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
关于postgres存储的资料非常多，往往配合着事务来一起来介绍，之前推荐的几本书中都有详细全面的讲解。
需要特别留意的是：
1. 磁盘上的多版本数据是"带内混合存储"的，这给pg带来了一系列特有的问题
1. pg这套存储是一个典型的行存系统（配合经典执行器结构），对于AP业务如何进行列存化改造是一个大主题

Questions:
1. null值是如何存储的？
    > 提示：标记位
1. 磁盘上的数据可能是压缩的，加载进内存后，什么时候解压的？
    > 提示：前边已经介绍了，另外可以在select*场景下gdb一下何时调用```pg_detoast_datum_packed()```
1. TupleSlot中的tts_values数组是干什么的？
    > 提示：看看和heaptuple.t_data的关系