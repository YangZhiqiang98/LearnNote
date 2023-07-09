# 23 | MySQL是怎么保证数据不丢的？

## binlog 的写入机制

1、事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。

2、一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。

3、系统给每个线程分配了一片 binlog cache 内存，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这大小，就要暂存到磁盘。

4、事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。

5、每个线程有自己 binlog cache，但是共用同一份 binlog 文件。

6、下图中的 write, 将日志写入到文件系统的 page cache，在内存中，所以速度很快；fsync 将数据持久化到磁盘，占用磁盘 IOPS。





![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230709170007.png)



> 参数 sync_binlog
>
> - sync_binlog = 0, 表示每次提交事务都只 write ，不 fsync
> - sync_binlog = 1, 表示每次提交事务都会执行 fsync
> - sync_binlog = N(N > 1), 表示每次提交事务都只 write，但累计 N 个事务后才 fsync。
>
> IO 瓶颈的场景中，将 sync_binlog 设置成一个较大的值，可以提升性能。但是对应风险就是，如果主机异常重启，会丢失最近 N 个事务的 binlog 日志。



## redo log 的写入机制

事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 的。



![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230709171822.png)



> innodb_flush_log_at_trx_commit 参数控制 redo log 的写入策略
>
> - innodb_flush_log_at_trx_commit  = 0， 表示每次提交事务都只是把 redo log 留在 redo log buffer 中，redo log buffer 是所有线程共用的。（上图红色部分）
> - innodb_flush_log_at_trx_commit  = 1， 表示每次提交事务时都将 redo log 直接持久化到磁盘。（上图绿色部分）
> - innodb_flush_log_at_trx_commit  = 2， 表示每次提交事务时都只是把 redo log 写到 page cache。（上图黄色部分）



InnoDB 的后台线程每隔 1 秒。就会把 redo log buffer 中的日志调用 write 写到文件系统的 page cache ，然后调用 fsync 持久化到磁盘 。



> PS: 事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。



一个没有提交的事务的 redo log 写入到磁盘的三种场景

- 后台线程每秒一次的轮询操作
- redo log buffer 占用空间达到 innodb_log_buffer_size 一半的时候，后台线程主动写盘。（注意，由于事务没有提交，这个写盘仅会写入到 page  cache）
- 并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。innodb_flush_log_at_trx_commit = 1 时，把 redo log buffer 里的日志全部持久化到磁盘。

### 组提交机制（group commit）

目的：节约磁盘 IOPS。提高 MySQL TPS



日志逻辑序列号（log sequence number，LSN）：LSN 是单调递增的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length。

LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log。





![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230709173445.png)



上图是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。

- trx1 是第一个到达的，会被选为这组的 leader；
- 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160；
- trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘；
- 这时候 trx2 和 trx3 就可以直接返回了。



在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 fsync 越晚调用，组员可能越多，节约 IOPS 的效果就越好。为了让一次 fsync 带的组员更多，MySQL 有一个很有趣的优化：拖时间。



两阶段提交的细化过程如下图：

![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20230709175057.png)



第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗。但是一般第 3 步执行的很快，导致 binlog write 和 fsync 间隔很短，binlog 组提交的效果不如 redo log 的组提交效果好。



> 提升 binlog 组提交的效果，可以通过设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count，这两个参数是或的关系。满足一个条件就会 fsync。
>
> - binlog_group_commit_sync_delay ：表示延迟多少微秒后才调用 fsync;默认为 0
> - binlog_group_commit_sync_no_delay_count：表示累积多少次以后才调用 fsync。默认为 0
>
> 这两个参数是基于额外的故意等待来实现的，会阻止客户端的响应。没有丢失数据的风险。



WAL 机制主要得益于两个方面：

1、redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；

2、组提交机制，可以大幅度降低磁盘的 IOPS 消耗。





## MySQL 出现 IO 性能瓶颈

1、设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。

2、将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。

3、将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。







## 数据库的 crash-safe



实际上数据库的 crash-safe 保证的是：

1、如果客户端收到事务成功的消息，事务就一定持久化了；

2、如果客户端收到事务失败（比如主键冲突、回滚等）的消息，事务就一定失败了；

3、如果客户端收到“执行异常”的消息，应用需要重连后通过查询当前状态来继续后续的逻辑。此时数据库只需要保证内部（数据和日志之间，主库和备库之间）一致就可以了。