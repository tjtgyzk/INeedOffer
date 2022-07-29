> MySQL日志主要包括：错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。
>
> 其中，比较重要的还要属二进制日志 `binlog`（归档日志）和事务日志 `redo log`（重做日志）和 `undo log`（回滚日志）。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201803585.png)

# redo log：持久性的保证

- `redo log`（重做日志）是`InnoDB`存储引擎独有的，它让`MySQL`拥有了崩溃恢复能力。
- 比如 `MySQL` 实例挂了或宕机了，重启时，`InnoDB`存储引擎会使用`redo log`恢复数据，保证数据的持久性与完整性。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201803439.png)

- `MySQL` 中数据是以页为单位，你查询一条记录，会从硬盘把一页的数据加载出来，加载出来的数据叫数据页，会放入到 `Buffer Pool` 中。
- 后续的查询都是先从 `Buffer Pool` 中找，**没有命中再去硬盘加载**，减少硬盘 `IO` 开销，提升性能。
- 更新表数据的时候，发现 `Buffer Pool` 里存在要更新的数据，就**直接在 `Buffer Pool` 里更新**。
- 然后会把**“在某个数据页上做了什么修改”**记录到**重做日志缓存（`redo log buffer`）**里，接着刷盘到 `redo log` 文件里。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201805278.png)

## 刷盘策略

`InnoDB` 存储引擎为 `redo log` 的刷盘策略提供了 `innodb_flush_log_at_trx_commit` 参数，它支持三种策略：

- **0** ：设置为 0 的时候，表示**每次事务提交**时**不进行刷盘操作**

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201813683.png)

为`0`时，如果**`MySQL`挂了或宕机**可能会有`1`秒数据的丢失。

- **1** ：设置为 1 的时候，表示**每次事务提交**时**都将进行刷盘操作**（默认值）

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201814923.png)

为`1`时， **只要事务提交成功**，`redo log`记录就一定在硬盘里，**不会有任何数据丢失**。

如果事务执行期间`MySQL`挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。

- **2** ：设置为 2 的时候，表示**每次事务提交**时都**只把 redo log buffer 内容写入 page cache**

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201815427.png)

为`2`时， 只要事务提交成功，`redo log buffer`中的内容**只写入文件系统缓存**（`page cache`）。

如果仅仅只是**`MySQL`挂了不会有任何数据丢失**，但是**宕机可能会有`1`秒数据的丢失**。



`InnoDB` 存储引擎有一个**后台线程**，每隔`1` 秒，就会把 `redo log buffer` 中的内容写到文件系统缓存（`page cache`），然后调用 `fsync` 刷盘。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201813662.png)

也就是说，**一个没有提交事务的 `redo log` 记录，也可能会刷盘。**



## 日志文件组

- 硬盘上存储的 `redo log` 日志文件不只一个，而是以一个**日志文件组**的形式出现的，每个的`redo`日志文件大小都是一样的。
- 比如可以配置为一组`4`个文件，每个文件的大小是 `1GB`，整个 `redo log` 日志文件组可以记录`4G`的内容。
- 它采用的是环形数组形式，**从头开始写，写到末尾又回到头循环写**，如下图所示。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201816998.png)

在个**日志文件组**中还有两个重要的属性，分别是 `write pos`、`checkpoint`

- **write pos** 是**当前记录的位置**，一边写一边后移
- **checkpoint** 是**当前要擦除的位置**，也是往后推移

每次刷盘 `redo log` 记录到**日志文件组**中，`write pos` 位置就会后移更新。

每次 `MySQL` 加载**日志文件组**恢复数据时，会清空加载过的 `redo log` 记录，并把 `checkpoint` 后移更新。

`write pos` 和 `checkpoint` 之间的**还空着的部分**可以用来写入新的 `redo log` 记录。

如果 `write pos` 追上 `checkpoint` ，表示**日志文件组**满了，这时候不能再写入新的 `redo log` 记录，`MySQL` 得停下来，清空一些记录，把 `checkpoint` 推进一下。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206201826274.png)

## 小结

 **只要每次把修改后的数据页直接刷盘不就好了，还有 `redo log` 什么事？**

- 实际上，数据页大小是`16KB`，刷盘比较耗时，可能就修改了数据页里的几 `Byte` 数据，**没有必要把完整的数据页刷盘。**
- 而且数据页刷盘是随机写，因为一个数据页对应的位置可能在硬盘文件的**随机位置**，所以性能很差。
- 如果是写 `redo log`，一行记录可能就占几十 `Byte`，只包含表空间号、数据页号、磁盘文件偏移 量、更新值，再加上是顺序写，所以刷盘速度很快。
- 所以用 `redo log` 形式记录修改内容，性能会远远超过刷数据页的方式，这也让数据库的并发能力更强。

# bin log：主从数据同步

- `redo log` 它是物理日志，记录内容是“在某个数据页上做了什么修改”，**属于 `InnoDB` 存储引擎。**
- 而 `binlog` 是逻辑日志，记录内容是**语句的原始逻辑**，类似于“给 ID=2 这一行的 c 字段加 1”，**属于`MySQL Server` 层。**
- 不管用什么存储引擎，只要发生了表数据更新，都会产生 binlog 日志。
- `binlog`会记录所有涉及更新数据的逻辑操作，并且是顺序写。

| 性质     | redo Log                                                     | bin Log                                                      |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 文件大小 | redo log 的**大小是固定的**（配置中也可以设置，一般默认的就足够了） | bin log 可通过配置参数max_bin log_size设置每个bin log文件的大小（但是一般不建议修改）。 |
| 实现方式 | redo log是**InnoDB引擎层实现的**（也就是说是 Innodb  存储引擎独有的） | bin log是  **MySQL  层实现的**，所有引擎都可以使用 bin log日志 |
| 记录方式 | redo log 采用**循环写**的方式记录，当写到结尾时，会回到开头循环写日志。 | bin log 通过**追加**的方式记录，当文件大小大于给定值后，**后续的日志会记录到新的文件上** |
| 使用场景 | redo log适用于**崩溃恢复**(crash-safe)（这一点其实非常类似与 Redis 的持久化特征） | bin log 适用于**主从复制和数据恢复**                         |

## 记录格式

`binlog` 日志有三种格式，可以通过`binlog_format`参数指定。

- **statement**
- **row**
- **mixed**

### statement

- 指定`statement`，记录的内容是`SQL`语句原文，比如执行一条`update T set update_time=now() where id=1`，记录的内容如下。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206211346976.png)

- 同步数据时，会执行记录的`SQL`语句，但是有个问题，`update_time=now()`这里会获取当前系统时间，**直接执行会导致与原库的数据不一致**。
- 为了解决这种问题，我们需要指定为`row`

### row

- 指定为`row`时，记录的内容不再是简单的`SQL`语句了，还包含操作的具体数据，记录内容如下。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206211347827.png)

- `row`格式记录的内容看不到详细信息，要通过`mysqlbinlog`工具解析出来。
- `update_time=now()`变成了具体的时间`update_time=1627112756247`，条件后面的@1、@2、@3 都是该行数据第 1 个~3 个字段的原始值（**假设这张表只有 3 个字段**）。
- 这样就能保证同步数据的一致性，通常情况下都是指定为`row`，这样可以为数据库的恢复与同步带来更好的可靠性。
- 但是这种格式，需要更大的容量来记录，比较**占用空间**，恢复与同步时会更消耗`IO`资源，**影响执行速度**。

### mixed

- 所以就有了一种折中的方案，指定为`mixed`，记录的内容是前两者的混合。
- `MySQL`会判断这条`SQL`语句是否可能引起数据不一致，如果是，就用`row`格式，否则就用`statement`格式。

## 写入机制

- 先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写到`binlog`文件中。
- 因为一个事务的`binlog`**不能被拆开**，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为`binlog cache`。
- 我们可以通过`binlog_cache_size`参数控制单个线程 binlog cache 大小，如果存储内容超过了这个参数，就要暂存到磁盘（`Swap`）。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206211605784.png)

- **上图的 write，是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快**
- **上图的 fsync，才是将数据持久化到磁盘的操作**

`write`和`fsync`的时机，可以由参数`sync_binlog`控制，默认是`0`。

- **为`0`的时候**，表示每次提交事务都只`write`，由系统自行判断什么时候执行`fsync`。虽然性能得到提升，但是机器宕机，`page cache`里面的 binlog 会丢失。
- **为`1`的时候**，表示每次提交事务都会执行`fsync`，就如同 **redo log 日志刷盘流程** 一样。
- **为`N`（N>1）的时候**，表示每次提交事务都`write`，但累积`N`个事务后才`fsync`。

# bin log和redo log的两阶段提交

- `redo log`（重做日志）让`InnoDB`存储引擎拥有了**崩溃恢复**能力。

- `binlog`（归档日志）保证了`MySQL`**集群架构的数据一致性**。

- 它们都属于**持久化的保证**，但是侧重点不同。

在执行更新语句过程，会记录`redo log`与`binlog`两块日志，以基本的事务为单位，`redo log`**在事务执行过程中可以不断写入**，而`binlog`**只有在提交事务时才写入**，所以`redo log`与`binlog`的写入时机不一样。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206211609450.png)

## 存在的问题

- 如果在写redo log时成功写入修改，但在写入bin log时发生了异常。

- 此时主库恢复时使用redo log，成功恢复了这次修改。但从库使用bin log恢复数据时，会丢掉这一次的更新。导致数据不一致

## 两阶段提交

为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用**两阶段提交**方案。

- 将`redo log`的写入拆成了两个步骤`prepare`和`commit`，这就是**两阶段提交**。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206211632036.png)

- 使用**两阶段提交**后，写入`binlog`时发生异常也不会有影响，因为`MySQL`根据`redo log`日志恢复数据时，发现`redo log`还处于`prepare`阶段，并且没有对应`binlog`日志，就会**回滚**该事务。
- 如果`redo log`在`commit`阶段发生异常，虽然`redo log`是处于`prepare`阶段，但是能通过事务`id`找到对应的`binlog`日志，所以`MySQL`认为是完整的，就会提交事务恢复数据。



# undo log：回滚，原子性的保证

- `undo` 顾名思义，就是没有做，没发生的意思。`undo log` 就是没有发生事情（**原本事情是什么**）的一些日志
- 如果想要保证事务的原子性，就需要在异常发生时，对已经执行的操作进行**回滚**，在 MySQL 中，恢复机制是通过 **回滚日志（undo log）** 实现的
- 所有事务进行的修改都会**先记录到这个回滚日志**中，然后再执行相关的操作。
- **回滚日志**会**先于数据**持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

另外，`MVCC` 的实现依赖于：**隐藏字段、Read View、undo log**。在内部实现中，`InnoDB` 通过数据行的 `DB_TRX_ID` 和 `Read View` 来判断数据的可见性，如不可见，则通过数据行的 `DB_ROLL_PTR` 找到 `undo log` 中的历史版本。每个事务读到的数据版本可能是不一样的，在同一个事务中，用户只能看到该事务创建 `Read View` 之前已经提交的修改和该事务本身做的修改

# 总结

- MySQL InnoDB 引擎使用 **redo log(重做日志)** 保证事务的**持久性**，使用 **undo log(回滚日志)** 来保证事务的**原子性**。

- `MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据**一致性**。