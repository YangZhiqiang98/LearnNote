## 02 | 日志系统：一条SQL更新语句是如何执行的？



查询语句的那套流程，更新语句也会走一遍。

<img src="https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230624231756.png" alt="img" style="zoom: 25%;" />



更新流程中和查询不一样的是，更新流程中涉及了两个重要的日志模块。`redo log` (重做日志) 和 `binglog`（归档日志）。



### redo log - InnoDb 特有

作用：用来记录对磁盘页中数据的变动。

特点：物理日志，大小固定，循环写入，当Mysql意外宕机后，记录好的 redo log 在 Mysql 重启后可根据 redo log 中的记录继续向磁盘中写入数据，这个能力被称之为：crash-safe。



WAL : Write-Ahead-Logging, 即先写日志，再写磁盘。

> 日志也是在磁盘上，不过是顺序 IO。后再找机会持久化随机 IO 写回到磁盘。



当有记录需要更新时，InnoDB 先把记录写到 redo log,并更新内存。之后，在系统比较空闲的时候，将记录更新到磁盘。



InnoDB 的 redo log 是 **固定大小** 的，且 **循环写入**。

> 修改如下两个参数，需要重启服务。
>
> [`innodb_log_file_size`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_log_file_size) : 设置 redo log 大小，默认值 48 MB。
>
> [`innodb_log_files_in_group`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_log_files_in_group)：设置 redo log 个数，默认值 2。
>
> 官网：https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html#innodb-redo-log-file-reconfigure



redo log 的写入过程如下：



<img src="https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230624231752.png" alt="img" style="zoom: 50%;" />





redo log 有两个指针，write pos 和 checkpoint。 write pos 是当前记录的位置，一边写一边后移，checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。write pos 和 checkpoint 之间的是还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示 redo log 满了，此时，不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。



**innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。**

### bin log - server 层

 归档日志 binlog 是所有 Mysql 引擎通用的日志系统，用来记录更新逻辑。

 **特点：**binlog 是逻辑日志，追加写入不会覆盖以前的日志。



**sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。这个参数建议设置成 1，这样可以保证 MySQL 异常重启之后 binlog不丢失。**

### redo log 和 bin log 不同

这两种日志有以下三点不同。

1、redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

2、redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。

3、redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。





### 更新语句执行过程

 例：将id为2的数据行中的c字段加1。（InnoDB）

1、执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。

2、执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

3、引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。

4、执行器生成这个操作的 binlog，并把 binlog 写入磁盘。

5、执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。



下图为 update 语句的执行流程图，图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。



<img src="https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230624233653.png" alt="img" style="zoom:50%;" />

将 redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。



### 两阶段提交

为什么必须有“两阶段提交”呢？  之所以使用两阶段提交，目的是为了让 binlog 与 redo log在逻辑上处于一致，这样当需要通过 binlog 做数据恢复时保证数据的一致性和正确性。

