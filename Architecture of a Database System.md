# Architecture of a Database System



数据库系统是非常重要且复杂的系统，但是其架构方面的知识却不像其他重要的系统（例如操作系统，编译器等）一样为人所熟知。传统教材通常着重讲述数据库相关的算法和理论知识，很少涉及到系统开发和架构方面。论文 [[1\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#DBS-002) 使用流行的商业和开源数据库系统作为例子，着重论述（关系型）数据库系统的架构。尽管有些细节方面在这些年中发生了变化，但是大体结构和思路上并没有太多出入。

这篇论文内容比较多，暂时先不怎么引入自己的想法，只是摘录一些重点内容，以免以后彻底遗忘论文内容。

## 整体结构

数据库系统的整体结构如 [Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 所示。

[![fntdb07 1 1](http://hcoona.github.io/images/fntdb07-1-1.svg)fntdb07 1 1](http://hcoona.github.io/images/fntdb07-1-1.svg)

Figure 1. 数据库主要组件

以一次对数据库的请求为例：

1. 首先需要和数据库建立持续的连接，这部分由 [Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 中最上面的组件 Client Communications Manager 负责。通常数据库需要支持不同的协议，例如 ODBC 和 JDBC，TCP 和本地 Pipe。
2. 用户连接建立后，需要为其分配线程资源，这部分由 [Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 中左边的组件完成。通常 Admission Control 也在这个时期进行。
3. 接下来用户的请求进入数据库的核心部分，通过 [Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 中间的组件 Relational Query Processor 进行处理。
   1. 用户的 SQL 查询首先被解析成为内部表示形式，通常是对应于关系代数的表达式树。
   2. 接下来 SQL 查询会被进行优化，在此之前一般会有一个 Rewrite 的步骤对 Query 进行一些预处理以简化 Optimizer 的逻辑。
   3. 经过优化后的 SQL 查询可能包含多个 Operator，这些 Operator 运算的结果还需要组合和串联，这部分工作由 Plan Executor 来执行。
4. Operator 的执行需要数据库底层进行支持，这部分功能由 [Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 中最下面的组件 Transactional Storage Manager 负责。

## 进程模型

数据库是面向多用户的服务，需要具备同时服务多个用户的能力，这需要一些基础的并行执行组件。数据库一般有自己的进程抽象，这是由于大部分数据库需要支持不同的运行环境，而一些早期操作系统对于线程的支持不够好，所以需要数据库自己进行适配。这里使用进程这一名词是因为可能数据库系统可能跨越多个计算节点。

一般数据库协议采用长连接模型，连接建立后会有对应关联的 Session 信息，维护了当前连接执行的命令的上下文环境，例如是否正在处理 Transaction 等等。数据库系统一般会为一个客户端连接分配一个对应的 DBMS worker 进行管理。DBMS worker 需要占用一定的计算资源，其采用的进程模型有以下几种情况：

1. Process per DBMS worker
2. Thread per DBMS worker
   1. OS thread per DBMS worker
   2. DBMS thread per DBMS worker
      1. DBMS threads scheduled on OS process
      2. DBMS threads scheduled on OS threads
3. Process/thread pool
   1. DBMS workers multiplexed over a process pool
   2. DBMS workers multiplexed over a thread pool

**个人认为**，出现这么多模型的大部分原因是受历史因素拖累，只考虑现代操作系统（尤其是 Linux）的话，由于可以较好的支持大量 threads，采用这几种模型都是比较合理的：

1. DBMS threads scheduled on OS threads
2. DBMS workers multiplexed over a thread pool

有些时候，数据库还会采用自己的线程库，里面进一步封装了用户态线程（也叫做纤程 Fiber）。这样做的优势是可以进一步的减少线程上下文切换带来的开销，缺点是维护成本较高，DEBUG 工具和信息也比较少。

## 准入控制

进行准入控制有以下两方面的考虑因素：

1. 防止一个用户占用过多资源，影响其他用户使用系统
2. 拒绝用户访问没有权限访问的内容

数据库系统的准入控制一般有以下两个时机：

1. 当用户请求到达时进行准入控制，避免为无效的请求分配资源
2. 在执行查询计划时进行控制，因为直到这一时刻才能方便的汇总所有必要的信息，例如将要执行物理查询计划的节点的负载

## 并行处理架构：进程和内存的协作

数据库系统为了性能考虑，需要更多的考虑进程和内存的协作，常见的几种模型如下：

1. Shared Memory
2. Shared-Nothing
3. Shared-Disk
4. NUMA

目前主流的单机数据库系统都支持了 Shared Memory 模型，这样做比较容易达到更高的性能。分布式的数据库系统一般使用 Shared-Nothing 模型，这基本上符合当前的硬件能力，即硬件并不提供（像访问本地节点数据一样可靠的）访问远程节点数据（内存 / 硬盘）的能力。目前一些新的硬件技术可能会打破这一假设，也是目前比较热点的一些新尝试领域。

采用 Shared-Disk 模型的分布式数据系统也有不少，这里假设所有进程都能以相似的代价访问共享磁盘。一些设施确实可以做到这一点，例如 SAN（Storage Area Networks）。或者将更底层的软件系统视为共享磁盘使用，以此建立 Shared-Disk 模型的系统，例如 BigTable[[2\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#bigtable)。

NUMA 尽管最早是作为分布式模型提出的，但是并没有在这一领域得到广泛的应用，反而在单机多核架构上发挥了巨大的作用。个人认为这种架构关于访问速度上的假设虽然勉强成立，但是对于可靠性方面的假设却不成立，因此很难在分布式领域得到很好的应用。主流的数据库系统在单机上一般都考虑了 NUMA 架构的影响。

## 关系查询处理（Relational Query Processor）

一般而言，关系查询的处理可以视为是单用户单线程执行的任务。并发控制是在更下层进行处理的，对上层提供了几乎透明的接口。查询主要分为两大类：

1. DML（Data Manipulation Language），例如 SELECT/INSERT/UPDATE/DELETE
2. DDL（Data Definition Language），例如 CREATE TABLE/CREATE INDEX

### 查询解析和授权（Query Parsing and Authorization）

对于一个 SQL 语句，解析的主要工作是

1. 检查查询语句是否正确
2. 获取名字和引用信息，例如 `SELECT c1 FROM t1 JOIN t2 ON t1.id = t2.t1id` 中
   1. `t1` 是 Table 名称，但是需要规范化为 4 阶段名称 `server.database.schema.table`，从而精确的定位这个表
   2. `c1` 是个列名，这个列是存在于 `t1` 还是 `t2` 中，也需要在这个阶段进行处理和规范化
3. 将查询转换为内部表示形式，通常是关系代数对应的内部表示形式
4. 检查用户是否有权限执行这个查询

上面这些工作一般都需要和 Catalog Manager 进行协作以获取表相关的元信息。此外，有些运算符还需要根据这些元信息确定类型，例如 `(EMP.salary * 1.15) < 75000` 中的比较运算符是整数比较，浮点数比较，还是 Decimal 比较，就取决于 `EMP.salary` 的类型。

有些约束也可以在这一时刻进行检查，例如 `SET EMP.salary = -1` 如果具有约束 `EMP.salary > 0` 的话，就可以在这一时刻检查出来并拒绝。

### 查询重写（Query Rewrite）

尽管有些数据库系统会将查询重写合并到上面的查询解析模块，或者下面的查询优化模块中，但是逻辑上这还是一个比较独立的功能，其主要功能是：

1. 视图（View）展开

2. 常量计算，例如 `R.x < 10 + 2 + R.y` ⇒ `R.x < 12 + R.y`

3. Predicate 重写，例如

   1. `NOT Emp.salary > 1000000` ⇒ `Emp.salary ⇐ 1000000`（节约了一个 Operator）
   2. `Emp.salary < 75000 AND Emp.salary > 1000000` ⇒ `False`（这种情况一般出现于视图展开之后）
   3. `R.x < 10 AND R.x = S.y` ⇒ `R.x < 10 AND S.y < 10 AND R.x = S.y`（逻辑传递关系，有可能利用 `S.y` 上的索引）

4. 语义优化，例如（常见于视图展开后）

   `SELECT Emp.name, Emp.salary  FROM Dept  INNER JOIN Emp ON Emp.deptno = Dept.dno`

   由于根本就没用到 `Dept` 表，所以可以转化成 `SELECT Emp.name, Emp.salary FROM Emp`。

5. 子查询展开和其他启发式重写规则

   由于查询优化是比较复杂的问题（NP-hard[[3\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#optimizer-np-hard))，为了保证复杂性上界，在进行查询优化时，一般不会跨越查询单元进行优化。因此在这之前，如果可能的话，将嵌套子查询重写为一个查询，会对后面的查询优化有所帮助。一般而言，在这一阶段会将所有等价的查询重写为一个标准形式。

   此外还有一些基于启发式规则或者代价预估进行的优化。

### 查询优化（Query Optimizer）

查询优化的主要工作是生成查询计划，目前主流的方法是沿用 Selinger 等人在实现 System R 时使用的方法 [[4\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#system-r-optimizer)。生成的查询计划有多种表示方法。早期的数据库系统为了追求性能一般直接生成机器码；后来为了保证一定的可移植性，一般生成中间结果然后解释执行。

尽管大家都沿用了 Selinger 的方法，但是也做出了不少改进：

1. 计划空间（Plan space）。出于性能考虑，Selinger 通过以下两种手段来缩减计划空间以减少计算量：只针对左偏树（left-deep tree）进行优化，推后执行笛卡尔积（Cartesian product）。但是大多数现代的数据库系统都会对这两种情况进行考虑，即同时针对左偏树和右偏树（bushy tree）进行优化，提前考虑笛卡尔积。
2. 选择代价估计（Selectivity estimation）。Selinger 简单的通过索引的大小估计表的大小。现代系统一般通过采样获得直方图和其他统计信息进行估计。
3. 搜索算法（Search Algorithms）。一些商业数据库系统，特别是 Microsoft 和 Tandem，没有采用 Selinger 的方法，而是采用一种自顶向下的搜索策略 [[5\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#cascade-optimizer)。这种搜索策略有时可以减少优化器考虑的计划数量 [[6\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#top-down-opt)，但是代价是增加优化器的内存使用量。有些系统会在搜索大量表格时会退化到使用启发式规则进行搜索的策略。
4. 并发（Parallelism）。现今主流的商业数据库系统都在一定程度上支持了并发处理，并且大多支持了查询内并发。查询优化器需要知道如何在 CPU 之间，甚至机器之间调度这些 parallelized operators。一个简单的思路是采用 2 级调度策略，一层只考虑在机器之间进行调度，另一层就像传统的优化器一样，只考虑机器内部的 CPU 之间调度。一些商业数据库系统使用了这样的策略，但是另一些没有使用 2 级调度，而是综合考虑了网络拓扑和数据分布进行全局调度。
5. 自动调整（Auto-Tuning）。数据的特征发生变化时，进行优化的策略也需要相应发生改变。一些公司正在试图通过机器学习等手段进行数据库系统的自动调优。

数据库系统通常会缓存一些查询计划以减少重新计算的开销。但是值得注意的是，这些缓存应该在合适的时机失效，例如做出优化决策的假设已经不复存在的时刻。

### 查询执行（Query Executor）

查询计划一般是有向图的形式表示的 dataflow，将表和 operator 联系起来。目前大多数现代查询执行引擎都采用 iterator 模型：

```
class iterator {  iterator &inputs[];  void init();  tuple get_next();  void close(); }
```

每一个 operator 都继承自 iterator。这样一来，operator 只需关注自身逻辑即可，无需关注其上下游。

iterator 模型的一个特性是数据流和控制流的强耦合，这一模型简化了不少情况下问题的处理。每调用一次 `get_next()` 方法，当其返回时，就预示着数据到达，并且调用结束，这就是数据流和控制流强耦合的情况。这样做只需一个线程即可驱动整个查询计划的执行，并且无需考虑 operator 之间性能匹配的问题。由于不需要进行调度和阻塞式等待，这样做也能够较为容易的达到非常高的系统利用率。而并发执行和网络通信可以通过封装 exchange iterator 的方式来解决 [[7\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#exchange-op)。

存取数据时，有两种可行的方法。一种是数据正在 buffer pool 中，此时需要 pin buffer page，然后拿到这个 tuple 的引用进行使用，使用完毕后 unpin buffer page，这种称为 BP-tuples。另一种是将 tuple 复制出来使用，称为 M-tuples。尽管 M-tuples 比 BP-tuples 容易管理，但是效率却低很多，一般只用于特殊情况，例如长查询。

大多数情况下，诸如 INSERT/DELETE/UPDATE 之类的写请求对应的查询计划非常简单和直接，但是有些特殊情况需要非常小心，特别是涉及到读后写的情况。例如（Halloween problem）“给每个工资低于 $20K 的员工涨 10% 的工资”，首先通过索引找到了候选 tuple，然后执行更新。但是数据更新会导致索引更新，因此这个 tuple 执行更新语句过后如果仍然符合过滤条件，有可能会再被选中执行更新。解决的方法一种是使用临时表之类的技术彻底将读和写过程分离，另一种是使用 Multi-version 之类的技术避免读到更新后的结果。

### 访存方法（Access Methods）

[Figure 1](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#fig-1-1) 中最下面模块中的 Access Methods。个人感觉类似于 Plan Executor 和 Storage Engine 的交叉点，也是最底层访问数据的 Operator。

这里讨论了几个问题，其中一个是需要将 Scan 的参数传递给底层，这是出于以下考虑：

1. 需要利用索引
2. 可以批量的 pin/unpin 或者 copy/delete 符合条件的 tuples 以提升性能

此外还提到了 Row ID 的选择，尽管使用 Physical disk address 可以获得更高的性能，但是当 B+-tree 分裂或者 tuple 需要移动时就比较复杂。另一种做法就是 secondary index 使用 primary key 作为 Row Id。Oracle 干脆允许 tuple span pages，以避免移动 tuples。

### 数据仓库（Data Warehouses）

数据仓库的场景（也称为 OLAP 场景）和 OLTP 的场景比较不一样，上面提到的优化过程和执行引擎的讨论需要一定的扩展和修改才能在 OLAP 的场景下得到更好的性能。

1. **Bitmap Indexes**。有些列，例如性别，只有固定的取值可能，使用 Bitmap 索引可以得到更好的性能。
2. **Fast Load**。尽管以前 OLAP 可以一天导入一次，但是现在一般都希望能够更快。Bulk load 可以跳过 SQL 解析等步骤，直接操作底层存储引擎，获得更好的性能。
3. **Materialized Views**。可以创建物理视图，更新数据时同时更新基表和物理视图，用空间换取查询时间。
4. **OLAP and Ad-hoc Query Support**。有些数据仓库具有固定的查询场景，因此可以预测到需要执行的语句，例如周期性执行的查询。这些场景某些程度上可以通过 data cubes 进行支持，类似于预计算。但是对于 ad-hoc 查询的支持一直是一个难题。
5. **Optimization of Snowflake Schema Queries**。OLAP 数据库的表结构一般设计成雪花形状，针对这种 schema 可以进行一些优化。

特别的 Column Storage 可以在 OLAP 的场景下发挥巨大的作用。

### 数据库可扩展性（Database Extensibility）

1. UDT/UDF
2. JSON/XML
3. Full-Text Search

## 存储管理

### 空间控制（Spatial Control）

众所周知，顺序存取远快于随机存取。数据库系统应当对于如何排布数据具有控制权，而且数据库系统比底层的操作系统知道更多信息，因此会比操作系统做的更好。早期的数据库系统通过直接操作裸磁盘的方式达成这一目的。但是这种方式会独占磁盘，管理和恢复数据困难，不能无痛享受其他技术（例如 SAN，RAID 等等），因此渐渐被其他方式所取代。目前主流的方法是分配一个大文件，然后使用 `mmap` API 或者 Direct I/O，Concurrent I/O 之类的技术对其进行操作，模拟一块裸磁盘。

### 缓存控制（Temporal Control：Buffering）

除了需要控制数据的位置，数据库系统还需要控制数据合适被写入持久化存储设备，这出于以下两方面考虑：

1. 一些数据必须立即写入持久化设备以保证正确性，例如事务的 write ahead logging
2. 数据库根据业务数据做了大量的缓存管理方面的工作，不需要底层操作系统也做类似的事情带来额外的开销

### 缓存管理（Buffer Management）

数据库一般使用一块内存空间（frame）和硬盘上的内容进行一一映射，这个映射不涉及到数据内容的转换以避免额外的 CPU 开销。这一映射关系和元数据等信息会被管理起来，其中一个重要的信息是 dirty flag。如果这个 frame 被决定换出内存，此时根据 dirty flag 决定是否需要将 frame 写回磁盘。上面提到过的 pin/unpin 也会决定这个 frame 是否能够被换出到磁盘中。

frame 根据一定的替换策略进行加载和换出，这方面的算法是过去研究的重点，例如 LRU，CLOCK，LRU-2 等算法。

## 事务：并发控制和恢复

数据库系统中真正比较大块，拆分的不怎么清晰的地方就是事务存储管理，通常由相互纠缠的以下 4 个模块构成：

1. 并发控制中的 Lock Manager
2. 错误恢复中的 Log Manager
3. 用于分离 I/O 的 Buffer Pool
4. 在底层磁盘上管理数据的 Access Methods

### A Note on ACID

ACID 指的是数据库系统中的事务：

| Atomicity   | 一个事务造成的改动，要么全部生效，要么全部不生效             |
| ----------- | ------------------------------------------------------------ |
| Consistency | 不能违反 SQL 定义的约束                                      |
| Isolation   | 两个并发运行的事务之间不能相互影响                           |
| Durability  | 一个事务一旦成功，其造成的改动 —— 除非被另外的过程所改写 —— 不能丢失 |

### A Brief Review of Serializability

主要有三大类实现并发控制的方法：

1. Strict two-phase locking（2PL）
2. Multi-Version Concurrency Control（MVCC）
3. Optimistic Concurrency Control（OCC）

### Locking and Latching

数据库会自己实现一个锁控制系统，称为 Locking。数据库系统会自己维护一个 Lock Table 记录 Transaction/Lock/Object 之间的关系，这样当 Abort 一个 Transaction 的时候，就能将与其关联的所有 Lock 都释放掉。此外，由于加锁的顺序是由用户存取数据的顺序驱动的，因此死锁检测也是一个必不可少的工作。这种锁系统主要是针对 Transaction 的。

数据库系统也会在访问数据结构时使用更细粒度的锁对数据及数据结构进行保护，这种锁称为 Latching。一般来说 Latching 都是由操作系统或者硬件指令提供的基础设施，因此对于 Latching 的使用要避免出现死锁的情况。

ANSI SQL 规定了几种隔离性级别：

1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE

这几种隔离性级别都是在早期针对于使用锁进行隔离性控制提出的规范。现在由于 MVCC 和 OCC 的使用，一些数据库系统也提供了其他的隔离性级别：

1. CURSOR STABILITY
2. SNAPSHOT ISOLATION
3. READ CONSISTENCY

### Log Manager

建议阅读 ARIES 的论文 [[8\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#aries)，其中不仅叙述了使用 Logging 的方法，也讨论了其他实现方法。

一般数据库系统使用 Write-Ahead Logging（WAL）实现事务持久化，其要点在于以下 3 点规则：

1. 每个对数据页的修改都应该产生一条日志，这个日志必须在内存页落盘之前落盘
2. 日志必须有序落盘
3. 日志落盘后才能响应事务成功

尽管基本规则非常简单，但是实际实现为了获得更好的性能通常远比这些规则复杂。关键在于保持事务提交的 *fast path* 上高性能的同时，提供 Rollback/Abort 的一定性能。在考虑到特定场景优化后，日志系统将变得更为复杂。

一般数据库采用 *DIRECT, STEAL/NOT-FORCE* 这样的原则 [[9\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#log-opt)：

1. 数据对象原地更新
2. 即便数据页上含有未提交的事务写入的数据，unpinned frame 也可以被替换掉（此时如果含有脏数据，则将改动写回磁盘。可以这么做是因为可以用 undo 日志撤销 aborted 事务的改动）
3. 事务 commit 时，无需将数据页落盘（因为写了 redo 日志了）

另一方面，减少日志的大小也可以有效的增加日志系统的性能。物理操作通常含有更多和当前结构相关的数据，这些数据可以提高执行这一操作时的性能，但是写日志时可以只记录逻辑操作，抛弃这些结构相关的数据，以进一步减少日志大小，提升写入日志时的性能。例如在 ARIES 系统中，将物理操作记入 UNDO 日志中，将逻辑操作记入 REDO 日志中。

系统非正常停机后，再次启动时需要从日志中恢复系统的状态。使用 *recovery log sequence number* （recovery LSN），记录日志的顺序；使用 *checkpoint* 周期性的记录 recover LSN 以避免从太早的时间点开始恢复过程。最简单的办法就是把所有的数据页全都落盘，然后记录一个检查点，但是这样做性能太低了。ARIES 使用了一种比较聪明的办法，以避免等待所有的数据页落盘。（没写细节）

特别需要注意的是事务 Rollback 也需要写 WAL，如果此时磁盘空间不足的话就会导致 Rollback 卡住的情况。一般采用预留一部分空间的方法来避免这一情况。

### Locking and Logging in Indexes

在 Index 结构上的 Locking 和 Logging 可能有不同于 Transaction 的策略以优化性能。

#### Latching in B+-Trees

如果使用严格的两阶段锁来保证 SERIALIZABLE 一致性的话，可以选择将整个 B + 树都锁住（锁住根节点）。但是这样一来，两个完全没有交集的事务也不能够并行运行了。常见的有以下 3 种策略解决这一问题：

1. 保守策略（Conservative schemes）。只有确保对数据页的访问没有任何相互影响的时候，才允许并发访问同一个数据页。例如一个操作正在遍历这个数据页，而另一个操作想要在这个数据页中进行插入操作，这样两个操作是不允许并发执行的，因为可能插入操作会导致数据页分裂。这种策略对比其他 2 个比较新的策略有点过于保守了。
2. Latch-coupling schemes。在遍历数据时，访问数据节点前对其加锁（Latch），只有在获取到下一个要访问的节点的锁后，才释放当前节点的锁。
3. Right-link schemes。B + 树中的节点存有一个右向指针。遍历时不使用上面提到的 Latch-coupling 的策略，访问完一个节点之后就释放锁。由于存在这样一个右向指针，在遍历的时候就可以察觉到是否在此期间发生了树的分裂，并且可以正确的找到所需的下一个节点。

特别的，Latch-coupling 只适用于 B + 树，而 Right-link 策略比较通用 [[10\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#gist)。

#### Logging for Physical Structures

B + 树的分裂可以不用 UNDO，所以在记日志的时候可以标记 REDO only。这种思想也可以用于其他结构的变换上，例如文件的增长等等。

### Next-Key Locking: Physical Surrogates for Logical Properties

B + 树在实现 Serializability 事务隔离性级别时，有特殊的优化方法，这种优化方法时针对于解决 “Phantom” 不一致现象的。Phantom 现象是这样一种现象，假设有一个查询带有一个范围谓词（predicate），例如 `Name BETWEEN 'Bob' AND 'Bobby'`，同时另一个查询向这个范围内插入了一条新的记录，例如 `Name = 'Bobbie'`，那么第一个查询就有可能在执行的过程中看到不一致的结果。这是因为在锁定记录时，没有对查询的这个范围本身加锁，导致了两个实际上有重叠的查询请求并行执行出现的异常结果（违背 Serializability 隔离级别）。

解决这一问题的一个方法是使用谓词锁，但是这么做代价比较高，一方面是因为判断任意两个谓词是否有交集是比较困难的，另一方面是因为基于哈希的 Lock Table 也很难支持这样的操作。

在 B + 树中可以使用这样一种方法进行优化。每次进行查询时，除了锁住自己需要的范围以外，还需要额外的多锁住一个恰好超出范围上界的元素。显然这样的方法是奏效的，并且因为 B + 树的结构，找到这样的下一个元素是比较容易的。

这种方法的思想是，使用一个实际存在的物理对象来代替一个不容易实现的逻辑性质。在这个例子中，下一个元素代替了扩大谓词范围来判断谓词交集问题这一抽象概念。这种技巧应当为人所知，以便在恰当的时候使用。

### Interdependencies of Transactional Storage

本章的开头提到过 Transactional Storage 是数据库系统中，子模块之间深度交缠的一个巨大的子系统，在这一节中将讨论各个子模块之间的相互依赖。

只考虑并发控制和错误恢复机制的话，就会发现常见的错误恢复机制 WAL 依赖于并发控制的实现方法是使用 strict two-phase locking。如果使用 non-strict two-phase locking 的话，如果已经 drop 了 lock，那么在 rollback 时候进行 undo 的时候，可能没办法再拿到锁。

如果再把 Access Methods 考虑进来的话，事情就更复杂了。Access Methods 的高性能实现本身就比较复杂，考虑到并发控制和错误恢复机制时，必须和其特有的数据结构紧密结合。这就是为什么主流的数据库系统一般都只实现了 B + 树和堆文件，只有 PostgreSQL 实现了 GiST。而且每种数据结构都有其特有的并发控制或者是错误恢复机制的优化技巧，即便是最简单的堆文件也有这样的特殊技巧，而这些特殊技巧是无法应用到其他数据结构上的。

在 Access Methods 上实现的并发控制一般只是在基于锁的实现上做的比较好。其他的并发控制方法，例如 MMVC 和 OCC，其实并没有考虑 Access Methods 的特性。因此在同一个 Access Methods 混合使用多种不同的并发控制方法是困难的。

在 Access Methods 上实现错误恢复的逻辑是一个和整个子系统高度相关的事情。数据结构的变更，使用物理变更日志还是使用逻辑变更日志，这些决策都离不开整个子系统的各个细节。例如在 B + 树中，恢复和并发逻辑就是交织的。一方面，如果需要进行错误恢复，则需要知道 B + 树可能进入什么样的不一致状态，然后才能通过日志来保证 Atomicity。另一方面，例如 B + 树节点的分裂，其实就不需要记录 UNDO 日志，并不需要在 Rollback 的时候把分裂的节点再合并回去，这就需要日志系统支持 REDO-only 的能力。

尽管 Buffer Manager 相对而言比较独立，看似和其他几个模块关联比较松散，但是这是因为我们实现了 DIRECT, STEAL/NOT-FORCE 这样的性质。而支持这些性质实际上是依赖于其他几个模块的支持的，所以 Buffer Manager 也跟另外几个子模块耦合在了一起。

## 公共组件

### Catalog Manager

Catalog Manager 中存放了整个数据库系统中的元数据，例如用户，Schemas，表，列，索引，等等。过去实践中的一个重要的经验是，应当使用和访问一般数据相同的方法（即 SQL）来访问和修改这些元数据。这些基础的数据需要进行特殊的对待，这通常是为了性能而进行的考虑。通常使用 denormalized 形式直接将这些数据存放在内存中提供服务，相关的 SQL 语句也直接进行了预编译和缓存，甚至一些事务相关的操作也进行了特殊优化。

### Memory Allocator

传统的教科书对于 Memory Allocator 的讲解都集中在内存池的管理上。但是实际上，数据库系统也将 Memory Allocator 应用于很多其他方面，例如 Selinger 的查询优化 [[4\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#system-r-optimizer) 中进行动态规划时就会使用大量的内存，更别提 hashjoin 和 sort 这种吃内存的 operator 了。

另一方面，数据库系统中的 Memory Allocator 还会使用一种 *context-based* 技术来同时增加性能和 debug 能力，其提供的基本 API 如下：

- **使用给定的名字或者类型创建一个 Context**。这个 Context 可以给 Allocator 关于被分配的内存将要用于何种用途的额外附加信息，这样 Allocator 就可以选择更合适的策略进行内存分配。例如用于 Query Optimizer 的内存每次只需要增长一点点，用于 HashJoin 的内存就是一大片一大片分配的。
- **给指定的 Context 分配一块内存**。类似于 `malloc()` 函数，但是是从内部的内存池中分配的。
- **回收指定的 Context 已经分配的一块内存**。类似于 `free()` 函数。这种用法其实比较少见，一般都是一下就将整个 Context 给回收了。
- **回收指定的 Context 的所有内存**。
- **重置一个 Context**。这种情况也会回收这个 Context 已经分配出去的所有内存，但是会保留这个 Context 的元信息，所以接下来还可以继续使用这个 Context 进行内存分配。

这和传统的 jemalloc 之类的，目标是无缝替换 `malloc` 调用的内存分配器略有不同。一些场景下，例如 optimizer 可能需要构建出 plan tree，因此会多次分配小内存对象，但是在这个 phase 结束后，将整个 context 一起回收掉，而不是再去遍历数据结构去进行小心翼翼的 `free` 操作。

这种使用方式和 Garbage Collector（GC）有些相似，但是比 GC 能够提供更多的可控性。例如其还保留了 `free` 操作，而且通过 Context 在内存分配和回收上进行了很好的隔离，也能获得更好的本地性。

由于数据库系统的数据流天然就是分成多个阶段进行的，这种 Memory Allocator 的设计方式特别适合数据库系统使用。特别是有些场景下如果事先知道无需进行 `free` 操作，而是在最后将整个 Context 回收的话，就无需追踪每一块分配出去的内存的使用情况，（对于经常分配小对象的场景）可以节约很多开销。

### Disk Management Subsystems

主要解决两类问题。一个问题是如何将数据库系统的数据和分布在不同磁盘的多个文件建立对应关系，其中每个文件可能还有大小限制。另一个问题是如何处理不同磁盘设备的特性带来的优化问题。

文件的大小和存取性能可能受操作系统，文件系统，物理设备的限制。数据库系统的数据和文件的大小也不能完全匹配，有些大的表可能一个设备上的单一文件装不下（例如早期文件系统的 2GiB 限制），有些小的表可能需要合并放在一个文件内。

SCSI 磁盘的性能和其他特性，和其他看起来是个磁盘但是实际上不是磁盘的设备可能相差很多，例如 RAID，SAN 等等。即便是 RAID，RAID-0 和 RAID-5 的特性又会相差很多。

### Replication Services

做 Replication 的方法主要有三种，但是只有最后一种性能上比较令人满意：

1. **Physical Replication**，即定期全盘复制
2. **Trigger-Based Replication**，使用数据库的 Trigger 功能
3. **Log-Based Replication**，将数据库日志（一般指 Bin-Log）近实时的写入另一个位置，然后再进行恢复。这里又有 2 种做法，一种是从日志中重建 SQL 进行回放，另一种就是直接回放数据变更。前者通用性好，可以跨不同数据库 vendor 进行数据复制；后者性能更高。

### Administration, Monitoring, and Utilities

- Optimizer Statistics Gathering
- Physical Reorganization and Index Construction
- Backup/Export（Fuzzy dump + logging）
- Bulk Load（Manipulate underlying access methods）
- Monitoring, Tuning, and Resource Governers

## 标准实践

这里汇总一下论文中所有提到的，流行的数据库系统在上面各个章节讨论的话题上，所采用的实现方式。

### 进程模型的标准实践

#### Process per DBMS worker

IBM DB2 在不支持高质量的 OS Thread 的系统上默认使用 Process per DBMS worker 模式，在支持高质量 OS Thread 的系统上默认使用 Thread per DBMS worker。

Oracle 也是和 DB2 的选择一样，但是还额外支持 Process Pool。

PostgreSQL 在所有系统上都使用 Process per DBMS worker 模式。

#### Thread per DBMS worker

这里还有两种变体：使用 OS thread 或者使用 DBMS thread。

IBM DB2 在操作系统支持高性能 Thread 时默认使用 OS thread。MySQL 也使用 OS thread。

DBMS thread 是数据库系统在用户态自己实现的任务调度抽象，有时也叫 “纤程”（Fiber）。这里又有两种情况，一种是基于 OS Process 实现，另一种是基于 OS Thread 实现。Sybase 和 Informix 支持了基于 OS Process 的 DBMS thread。大部分数据库系统都是用 OS Process 来实现 DBMS thread，但是不是所有的系统都支持在 OS Process 之间迁移 DBMS thread。MS SQL Server 支持了 OS Thread 实现的 DBMS Thread，但是应用场景比较少。

#### Process/thread pool

使用 Process pool 要比 Process per DBMS worker 模式更节省内存，而且也容易 Port 到对 Thread 支持不好的 OS 上。Oracle 也将这种模式作为一种支持的选项，并且推荐在用户并发连接多时使用这种模式。Oracle 的默认模式是 Process per DBMS worker，这两种模式都能很容易的扩展到很多种类的操作系统上。

MS SQL Server 默认使用 Thread pool。

### 并行处理架构的标准实践

- **Shared-Memory**: 所有主流的商业数据库系统都支持这种模式，包括：IBM DB2，Oracle，MS SQL Server。
- **Shared-Nothing**: IBM DB2，Informix，Tandem，NCR Teradata 都支持这种模式；Greenplum 提供一个 PostgreSQL 支持 Shared-Nothing 模式的定制版本。
- **Shared-Disk**: 支持这种模式的有 Oracle RAC，Oracle RDB，IBM DB2 for zSeries

### 关系查询处理的标准实践

从粗粒度的架构来看，几乎所有的关系型数据库的查询引擎都和 System R 的原型 [[11\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#system-r) 差不多。这些年相关的进展主要集中在这个框架内怎么加速更多种类的查询和 Schema，主要的改进有以下几个方面：

1. 查询优化的搜索策略（top-down vs. bottom-up）
2. 查询执行的控制流模型（iterators + exchange operator vs. asynchronous producer/consumer）

在细粒度而言，不同的厂商有很多不同的做法，涉及到 optimizer，executor 和 access methods 的综合优化来达到更好的性能，尤其是涉及到不同的 workload 类型时，例如

1. OLTP
2. decision-support for warehousing
3. OLAP

具体的做法都是各个厂商的 “秘方”，唯一知道的就是大家做得都挺好的。

开源领域，PostgreSQL 使用了比较成熟的传统的 cost-based 的优化器，拥有一系列 execution 算法和很多商业产品中所没有的扩展功能。MySQL 的查询引擎就简单多了，基本上就是 nested-loop joins over indices。MySQL 的查询优化着力于分析型的查询，确保整个过程的轻量和高效，特别是 key/foreign-key joins，outer-join-to-join rewrite，以及只查询结果的前若干行的场景。

### 存储管理的标准实践

现在数据库系统对于底层存储的主要使用方式是这样的，直接在指定的磁盘上创建一个大文件，通过底层系统调用（例如 mmap）直接对这个文件进行操作。基本上数据库系统将这个文件视为一大块连续的数据库数据页的数组。

### 事务的标准实践

现如今，所有的产业级数据库系统都支持 ACID 事务，并且基本上都使用 WAL 来实现 Durability，使用 2PL 实现并发控制。PostgreSQL 是一个特例，只使用 MVCC 实现并发控制。Oracle 是一个在提供了 2PL 以外还提供 MVCC 提供其他弱一致性模型的先行者。采用 B+ 树来做索引也基本上时所有数据库系统的标准了。有些数据库系统或者通过直接提供，或者通过插件的形式，还提供了多维索引的能力。但是只有 PostgreSQL 通过其特有的 GiST [[10\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#gist) 提供了高并发的多维索引和全文索引。

MySQL 的独到之处在于其支持多种实现作为其底层存储，并且允许 DBA 为同一个数据库内的不同表指定使用不同的存储管理实现。MyISAM 只支持表级锁，但是对于读多请求的表现最好。InnoDB 提供了行级锁，适用于读写均衡的场景。但是两种存储引擎都没有实现 System R 著名的多级锁粒度支持 [[12\]](http://hcoona.github.io/Paper-Note/architecture-of-a-database-system/#granularity-locks)，因此在某些场景下两者表现得都不是很好，例如在混合了 scan 和 high-selectivity index access 的场景。

## References

- [1] HELLERSTEIN J M, STONEBRAKER M, HAMILTON J. Architecture of a Database System[J]. Foundations and Trends® in Databases, 2007, 1(2): 141–259.
- [2] CHANG F, DEAN J, GHEMAWAT S, et al. Bigtable: A Distributed Storage System for Structured Data[J]. 7th USENIX Symposium on Operating Systems Design and Implementation (OSDI), 2006: 205–218.
- [3] IBARAKI T, KAMEDA T. On the Optimal Nesting Order for Computing N-relational Joins[J]. ACM Trans. Database Syst., New York, NY, USA: ACM, 1984, 9(3): 482–502.
- [4] SELINGER P G, ASTRAHAN M M, CHAMBERLIN D D, et al. Access Path Selection in a Relational Database Management System[C]//Proceedings of the 1979 ACM SIGMOD International Conference on Management of Data. New York, NY, USA: ACM, 1979: 23–34.
- [5] GRAEFE G. The Cascades Framework for Query Optimization[J]. IEEE Data Eng. Bull., 1995, 18(3): 19–29.
- [6] SHAPIRO L D, MAIER D, BENNINGHOFF P, et al. Exploiting Upper and Lower Bounds In Top-Down Query Optimization[C]//Proceedings of the International Database Engineering &Amp; Applications Symposium. Washington, DC, USA: IEEE Computer Society, 2001: 20–33.
- [7] GRAEFE G. Encapsulation of Parallelism in the Volcano Query Processing System[C]//Proceedings of the 1990 ACM SIGMOD International Conference on Management of Data. New York, NY, USA: ACM, 1990: 102–111.
- [8] MOHAN C, HADERLE D, LINDSAY B, et al. ARIES: A Transaction Recovery Method Supporting Fine-granularity Locking and Partial Rollbacks Using Write-ahead Logging[J]. ACM Transactions on Database Systems, New York, NY, USA: ACM, 1992, 17(1): 94–162.
- [9] HAERDER T, REUTER A. Principles of Transaction-oriented Database Recovery[J]. ACM Comput. Surv., New York, NY, USA: ACM, 1983, 15(4): 287–317.
- [10] KORNACKER M, MOHAN C, HELLERSTEIN J M. Concurrency and Recovery in Generalized Search Trees[C]//Proceedings of the 1997 ACM SIGMOD International Conference on Management of Data. New York, NY, USA: ACM, 1997: 62–72.
- [11] ASTRAHAN M M, BLASGEN M W, CHAMBERLIN D D, et al. System R: Relational Approach to Database Management[J]. ACM Trans. Database Syst., New York, NY, USA: ACM, 1976, 1(2): 97–137.
- [12] GRAY J, LORIE R A, PUTZOLU G R, et al. Granularity of Locks and Degrees of Consistency in a Shared Data Base[C]//NIJSSEN G M. Proceeding of the {IFIP} Working Conference on Modelling in Data Base Management Systems. North-Holland, 1976: 365–394.