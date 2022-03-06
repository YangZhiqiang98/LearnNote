## InnoDB 体系架构

[脑图](https://www.processon.com/embed/604e0f8ff346fb348a8e366e)

### InnoDB 关键特性

- 插入缓冲（Insert Buffer）
- 两次写（Double Write）
- 自适应哈希索引（Adaptive Hash Index）
- 异步 IO（Async IO)
- 刷新邻接页（Flush Neighbor Page）



#### 插入缓冲

##### 1、Insert Buffer

插入缓冲**不是**缓冲池的一部分，虽然缓冲池中有 Insert Buffer 的信息， 但它是物理页的一部分。

对于**非聚集索引的<span style="color:red">插入或更新</span>操作**，不是每一次直接插入到索引中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer 对象中，然后再以一定的频率和情况进行 Insert Buffer 和辅助索引页子节点的 merge (合并) 操作，这时通常能将多个插入合并到一个操作中（因为在一个索引页中），提高了非聚集索引插入的性能。

然而 Insert Buffer 的使用要**同时满足**以下两个条件：

- 索引是辅助索引（Secondary Index）
- 索引不是唯一（unique）的

索引不能是唯一的，因为在插入缓冲时，数据库并不去查找索引页来判断插入的记录的唯一性。如果去查找肯定又会有离散读取的情况发生，从而导致 Insert Buffer 失去了意义。

存在的问题：在写密集的情况下，插入缓冲会占用过多的缓冲池内存（innodb_buffer_pool）,默认最大可以占用到 1/2 的缓冲池内存。

##### 2、Change Buffer

Insert Buffer 的升级。可以对 DML 操作 —— **Insert、Delete、Update** 都进行缓冲，它们分别是： Insert Buffer、Delete Buffer 、Purge Buffer。和 Insert Buffer 一样，适用对象依然是**非唯一的辅助索引**。



对于一条记录进行 UPDATE 操作可能分为两个过程：

- 将记录标记为已删除 ——Delete Buffer
- 真正将记录删除 ——Purge Buffer

InnoDB 提供了 innodb_change_buffering 来控制 Change Buffer 的选项。可选值有inserts、deletes、purges、changes、all（默认）、none。

InnodB 提供了 innodb_change_buffer_max_size  控制 Change Buffer 最大使用内存的数量，默认 25，最大有效值为 50。 

##### 3、Insert Buffer 的内部实现

全局 B+ 树。

##### 4、Merge Insert Buffer

Insert Buffer 中的记录何时合并到真正的辅助索引汇中呢？

- 辅助索引页被读取到缓冲池时
- Insert Buffer Bitmap 页追踪到该辅助索引页已无可用空间时
- Master Thread

#### 两次写

解决问题：当发生数据库宕机时，可能 InnoDB 存储引擎正在写入某个页到表中，这个页只写了一部分，这种情况称为**部分写失效（partial page write）**。

![InnoDB doublewrite 架构](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/InnoDB%20doublewrite%20%E6%9E%B6%E6%9E%84.png)

double write 由两部分组成：

- 内存中的 doublewrite buffer,大小为 2 MB。
- 物理磁盘上共享表空间中连续的 128 个页，即 2 个 区（extent)，大小同样为 2 MB。



在对缓冲池中的脏页进行刷新时，并不直接写磁盘，而是会通过 memcpy 函数将脏页先复制到内存中的 doublewrite buffer，之后通过 doublewrite buffer 再分两次，每次 1 MB 顺序地写入共享表空间的物理磁盘上，然后马上调用 fsync 函数，同步磁盘，避免缓冲写带来的问题。这个过程是顺序写入的，开销不是很大。在完成 doublewrite 的写入后，再将 doublewrite buffer 中的页写入到各个表空间中，此时写入则是离散的。

如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB 存储引擎可以从共享表空间中的 doublewrite 中找到该页的一个副本，将其复制到表空间文件中，再应用重做日志。

#### 自适应哈希索引

InnoDB 存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引（Adaptive Hash Index, AHI ）。InnoDB 存储引擎会自动根据访问的频率和模式来自动的为某些热点页建立哈希索引。

AHI 的要求：

- 对页的连续访问模式必须是一样的。例如对于 （a,b）联合索引，访问模式可以是 `where a =xxx` 和`where a = xx and b = xxx`，若交替进行上述两种查询，InnoDB 不会为该页构造 AHI。
- 只能用来搜索等值的查询。



#### 异步 IO

在 InnoDB 存储引擎中，read ahead 方式的读取都是通过 AIO 完成，脏页的刷新，即磁盘的写入操作则是全部由 AIO 完成。

> BIO （Blocking I/O）：同步阻塞I/O模式
>
> NIO （New I/O）：同步非阻塞I/O模式
>
> AIO （ Asynchronous I/O）：异步非阻塞I/O模型



#### 刷新邻接页

工作原理：当刷新一个脏页时，InnoDB 存储引擎会检测该页所在区（extent）的所有页，如果是脏页，则一起进行刷新。 通过 AIO 可以将多个 IO 写入操作合并为一个 IO 操作，这种工作机制在传统机械硬盘下有着显著的优势。但需要考虑以下两个问题：

- 是不是可能将不怎么脏的页进行了写入，而该页之后又会很快变成脏页。
- 固态硬盘具有较高的 IOPS，是否需要这个特性？

为此，InnoDB 提供了 innodb_flush_neighbors  用来控制是否开启刷新邻接页特性，建议机械硬盘下开启，固态硬盘下关闭。



> IOPS (Input/Output Per Second)即每秒的输入输出量(或读写次数)，是衡量磁盘性能的主要指标之一。IOPS是指单位时间内系统能处理的 I/O 请求数量，一般以每秒处理的 I/O 请求数量为单位，I/O 请求通常为读或写数据操作请求。 



## 文件

- 参数文件：告诉 MySQL 实例启动时哪里可以找到数据库文件，并且指定某些初始化参数，这些参数定义了某种内存结构的大小等设置。
- 日志文件：用来记录 MySQL 实例对某种条件做出相应时写入的文件，如错误日志文件、二进制日志文件、慢查询文件、查询日志文件等。
- socket 文件：当用 UNIX 域套接字方式进行连接时需要的文件。
- pid 文件：MySQL 实例的进程 ID 文件。
- MySQL 表结构文件：用来存放 MySQL 表结构定义的文件。
- 存储引擎文件：每个存储引擎都会有自己的文件来保存各种数据。这些存储引擎真正存储了记录和索引等数据。



### 参数文件

当 MySQL 实例启动时，数据库会先去读一个配置参数文件，用来寻找数据库的各种文件所在位置以及指定某些初始化参数。在默认情况下，MySQL 实例会按照一定的顺序在指定的位置进行读取，用户只需通过命令 `mysql --help |grep my.cnf`来寻找即可。

和 Oracle 实例不同，MySQL 实例可以不需要参数文件，这时所有的参数值取决于编译 MySQL 时指定的默认值和源代码中指定参数的默认值。

MySQL 数据库的参数文件是以文本方式进行存储的。



#### 什么是参数

通过命令 `show variables;` 查看数据库中的所有参数，也可以通过 LIKE 来过滤参数名。

```mysql
show variables;
show variables like 'innodb_%';
```



#### 参数类型

- 动态（ dynamic）参数：可以在 MySQL 实例运行中进行更改
- 静态（static）参数：在整个实例生命周期内不得进行更改

有些动态参数只能在会话中进行修改，如 autocommit；而有些参数修改完后，在整个实例生命周期中都会生效，如 binglog_cache_size；而有些参数即可以在会话中又可以在整个实例的生命周期内生效，如 read_buffer_size。

```mysql
set @@session.read_buffer_size=1048576;
set @@global.read_buffer_size=1048576;
```



### 日志文件

日志文件记录了影响 MySQL 数据库的各种类型活动。

- 错误日志（error log）
- 二进制日志（bin log）
- 慢查询日志（slow query log）
- 查询日志（log）



#### 错误日志

错误日志对 MySQL 的启动、运行、关闭过程进行了记录。该文件不仅记录了所有的**错误信息，也记录一些警告信息或正确的信息**。可通过命令 `show variables  like  'log_error';`

![image-20210901201240037](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210901201240037.png)



可以看到错误文件的路径和文件名。默认情况下，错误文件文件名为服务器的主机名。当出现 MySQL 数据库不能正常启动时，第一个必须查找的文件应该就是错误日志文件。有时用户可以直接在错误日志文件中得到**优化**的帮助，因为有些警告（warning）很好的说明了问题所在。



#### 慢查询日志

**慢查询日志可以定位到可能存在问题的 SQL 语句**，从而进行 SQL 语句层面的优化。

例如，可以在 MySQL 启动时设置参数 `long_query_time`（默认值为 10 秒），将运行时间**超过**该参数值的所有 SQL 语句都记录到慢查询日志文件中。

默认情况下，慢查询日志是关闭的。

查看

```mysql
-- 查看慢查询阈值
show variables like  'long_query_time';
-- 查看慢查询是否开启
Show variables like 'slow_query_log';
-- 查看慢查询日志输出文件路径
Show variables like 'slow_query_log_file';
-- 查看慢查询日志输出方式
Show variables like 'log_output';
```

配置

```mysql
-- 打开慢日志记录
set global slow_query_log=‘ON’;
-- 将慢日志记录到 mysql.slow_log 表
set global log_output=‘TABLE’;
-- 设置超过5秒的查询为慢查询（如果失败，手动修改my.ini文件，添加long_query_time=5 重启mysql 即可）
set global long_query_time= 5; 
```

当日志文件很大时，难以简单和直观的分析文件，可以通过 mysqldumpslow 命令。



和慢查询日志有关的参数

- `log_queries_not_using_indexes`：如果运行的 SQL 语句没有使用索引，则这条 SQL 也会被记录到慢查询日志文件。（默认关闭）

- `log_throttle_queries_not_using_indexes`：每分钟记录到 slow log 的且未使用索引的 SQL 语句次数，默认为 0，表示没有限制。

- `log_output`：指定慢查询输出的格式，默认为 FILE，可设置为 TABLE，记录到 mysql 架构下的 slow_log 表。

#### 查询日志

查询日志记录了所有对 MySQL 数据库请求的信息，无论这些请求是否得到了正确的执行。默认文件名为：主机名.log。默认关闭。

```mysql
-- 查看查询日志是否开启
show variables like 'general_log';
-- 查询日志输出文件路径
show variables like 'general_log_file';
-- 将查询日志写入到 mysql 架构下的 general_log 表中
set global log_output= 'TABLE';
```



#### 二进制日志

二进制日志（binary log） 记录了对 MySQL 数据库执行**更改**的所有操作，但是不包括 SELECT 和 SHOW 这类操作，因为这类操作对数据本身并没有修改。



```mysql
-- 查看当前使用的二进制日志文件
show master status;
-- 查看二进制日志文件事件
show binlog events in 'binlog.000054';
```

![image-20210928223433266](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210928223433266.png)

这里的 binlog.000054 即为二进制日志文件。bin_log.index 为二进制索引文件，用来存储过往产生的二进制日志序号。

**作用**：

- 恢复（recovery）：某些数据的恢复需要二进制日志
- 复制（replication）
- 审计：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

**参数**：

- **max_binlog_size**（默认 1073741824，代表 1 G）

   指定了单个二进制日志文件的最大值，如果超过该值，则产生新的二进制日志文件，后缀名 + 1，并记录到 .index 文件。

- **binlog_cache_size**

  当使用事务的表存储引擎时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓存中的二进制日志写入二进制日志文件，而缓冲的大小由 bing_cache_size 决定，默认为 32K。此外，binlog_cache_size 是**基于会话（session）**的，当一个线程开始一个事务时，MySQL 会自动分配一个大小为 binlog_cache_size 的缓存。当一个事务的记录大于设定的 binlog_cache_size 时，MySQL 会把缓冲中的日志写入到一个临时文件中。

  ```mysql
  -- 查看 binlog_cache_use、binlog_cache_disk_user 状态，用来判断当前 binlog_cache_size 的设置是否合适
  show global status like 'binlog_cache%';
  ```

  +---------------------+-----+
  |Variable_name        |Value|
  +---------------------+-----+
  |Binlog_cache_disk_use|0    |
  |Binlog_cache_use     |2    |
  +---------------------+-----+

  binlog_cache_disk_use ： 记录使用临时文件写二进制日志的次数。

  binlog_cache_user : 记录使用缓冲写二进制日志的次数。

- **sync_binlog= [N]**

   默认情况下，二进制日志并不是每次写的时候同步到磁盘（可以理解为缓冲写）。因此，当数据库所在操作系统发生宕机时，可能会有最后一部分数据没有写入二进制日志文件中。**参数 sync_binlog = [N] 表示每写缓存多少次就同步到磁盘**。 sync_binlog= 1 表示采用同步写磁盘的方式来写二进制日志（默认）。

  在**之前的版本（8.0之前）**中，即使将 sync_binlog 设为 1，还是不能保证二进制日志和表内容的一致性。如使用 InnoDB 存储引擎，在一个事务发出 COMMIT 动作之前，由于 sysnc_binlog , 因此会将二进制日志立即写入磁盘，如果这时发生了宕机，下次启动时，由于 COMMIT 操作没有发生，这个事务会被回滚，但是二进制日志文件记录了该事务信息，不能被回滚。这个问题可以通过将参数 innodb_support_xa = 1 来解决。

  **在 8.0 版本，innodb_support_xa 参数被移除，8.0 之后始终启用对 XA 事务中两阶段提交的 InnoDB 支持**。InnoDB 对 XA 事务中两阶段提交的支持确保了二进制日志和 InnoDB 数据文件的同步。但是，MySQL 服务器还应配置为在提交事务之前将二进制日志和 InnoDB 日志同步到磁盘。默认情况下，InnoDB 日志是同步的，sync_binlog = 1 确保二进制日志是同步的。

- **binlog_format**

  设置记录二进制日志的格式。参数可设为 STATEMENT、ROW（默认）、MIXED。

  - STATEMENT 记录的是日志的逻辑 SQL语句。

  - **ROW 记录表的行更改情况。(默认)**

  - 在MIXED 格式下，MySQL 默认采用 STATEMENT 格式进行二进制日志文件的记录，但在一些情况下会使用 ROW 格式，
    - 表的存储引擎为 NDB，对表的 DML 操作会以 ROW 格式记录。
    - 使用了 UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT() 等不确定函数
    - 使用了 INSERT DELAY 语句
    - 使用了用户定义函数
    - 使用了临时表（temporary table）

查看二进制日志文件内容

```cmd
E:\Server\mysql\data>mysqlbinlog binlog.000054
mysqlbinlog: [ERROR] unknown variable 'default-character-set=utf8mb4'.

E:\Server\mysql\data>mysqlbinlog -no-defaults binlog.000054
```



### pid 文件

当 MySQL 实例启动时，会将自己的进程 ID 写入到一个文件中--该文件即为 pid 文件。该文件可由参数 pid_file 参数控制，默认位于数据库目录下，文件名为 主机名.pid。

```mysql
show variables like 'pid_file';
```

![image-20210929203724290](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210929203724290.png)



![image-20210929203607730](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210929203607730.png)



### ~~表结构定义文件（已移除）~~

每个表都有一个以 frm 为后缀的文件，记录里该表的表结构定义。

frm 还用来存放视图的定义。

### InnoDB 存储引擎文件

之前都是 MySQL 数据库本身的文件，与存储引擎无关。

#### 表空间文件

InnoDB 采用将存储的数据按表空间进行存放的设计。在默认配置下会有一个初始大小为12 MB，名为 ibdata1的文件。

```mysql
show variables like 'innodb_data_file_path';
```

![image-20210929204646822](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210929204646822.png)

可通过参数设置，如：

```mysql
innodb_data_file_path=/db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
```

 将 /db/ibdata1 和 /dr2/db/ibdata2 两个文件用来组成表空间。若这两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。autoextend 表示文件可以自增长。



设置了 innodb_data_file_path 参数后，所有基于 InnoDB 存储引擎的表的数据都会记录到该共享表空间中。

若设置了参数 innodb_file_per_table（默认为 ON），则用户可以将每个基于 InnoDB 存储引擎的表产生一个独立表空间。通过这样的方式，用户不用将**所有的数据**都存放在默认的表空间中。

![image-20210929205431741](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210929205431741.png)



需要注意的是，这些**单独的表空间文件仅存储该表的数据、所用和插入缓冲 BITMAP 等信息，其余信息还是存放在默认的表空间中。**



#### 重做日志文件

在默认情况下，在 InnoDB 存储引擎的数据目录下会有两个名为 ib_logfile0 和 ib_logfile1 的文件。这两个文件即是 重做日志文件（redo log file）。记录了对于 InnoDB 存储引擎的事务日志。

当实例或介质失败（media failure）时，如掉电导致实例失败，InnoDB 存储引擎会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性。

每个 InnoDB 存储引擎至少有 1 个重做日志文件组（group），每个文件组下至少有 2 个重做日志文件。为了高可靠性，可设置多个镜像日志组（mirrored log groups） ，将不同的文件组放在不同的磁盘上，以此提高重做日志的可用性。在日志组汇总每个重做日志文件的大小一致，并以循环写入的方式来运行。

InnoDB 先写 重做日志文件 1 ，当达到文件最后时，切换至重做日志文件 2，当重做日志文件 2 被写满时，再切换到重做日志文件 1 中。

下图是一个拥有 3 个 重做日志文件的重做日志文件组。



<img src="https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%84.png" alt="日志文件组"  />



##### 相关属性

1、[innodb_log_file_size](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_log_file_size)

指定每个重做日志文件的大小。默认 48 MB，最大 512 GB。

2、innodb_log_files_in_group

指定日志文件组中重做日志文件的数量，默认为 2 。

3、innodb_log_group_home_dir

指定日志文件组所在路径，默认为 ./ ，表示在 MySQL 数据库的数据目录下。

4、~~innodb_mirrored_log_groups~~ （RDS 中没有找到）

指定日志镜像文件组的数量，默认为 1



#### bin log 和 redo_log 区别

- **记录的范围不同**。bin log 会记录所有与 MySQL 数据库有关的日志记录，包括InnoDB、MyISAM、Heap 等其他存储引擎的日志。而 InnoDB 的重做日志只记录有关该存储引擎本身的事务日志。
- **记录的内容不同**。bin log 记录的是关于一个事务的具体操作内容，即该日志是逻辑日志。InnoDB 的重做日志文件记录的是关于每个页（page）的更改的物理情况。
- **写入的时间不同**。bin log 仅在事务提交前进行提交，即只写磁盘一次，不论此时该事务多大。而在事务进行的过程中，却不断有重做日志条目（redo entry） 被写入到重做日志文件中。
- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。追加写是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。
- binlog 可以作为恢复数据使用，主从复制搭建，redo log 作为异常宕机或者介质故障后的数据恢复使用。

> redo log 和 binlog 区别：
>
> 1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
> 2. redo log 是物理日志，记录的是在某个数据页上做了什么修改；binlog 是逻辑日志，记录的是这个语句的原始逻辑。
> 3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。追加写是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。





![重做日志条目结构](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E6%9D%A1%E7%9B%AE%E7%BB%93%E6%9E%84.png)

- redo_log_type 占用 1 字节，表示重做日志的类型

- space 表示表空间的 ID，但采用压缩的方式，因此占用的空间可能小于 4 字节
- page_no 表示页的偏移量，同样采用压缩的方式
- redo_log_body 表示每个重做日志的数据部分，恢复时需要调用相应的函数进行解析





写入重做日志文件的操作不是直接写，而是先写入一个重做日志缓冲（redo log buffer）中，然后按照一定的条件顺序地写入日志文件。

![重做日志写入过程](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E5%86%99%E5%85%A5%E8%BF%87%E7%A8%8B.png)



从重做日志缓冲往磁盘写入时，是按 512 个字节，也就是一个扇区的大小。因此扇区是写入的最小单位，因此可以保证写入必定是成功的，因此在重做日志的写入过程中不需要有 doublewrite。



哪些条件会触发日志缓冲写入磁盘上的重做日志文件呢？

1、主线程 **每秒** 会将重做日志缓冲写入到磁盘的重做日志文件中，不论事务是否已经提交。

2、参数 innodb_flush_log_at_trx_commit 控制，表示在提交操作时，处理重做日志的方式。有效值有以下几种：

- 0：代表当提交事务时，并不将事务的重做日志写入到磁盘上的日志文件，而是等待主线程每秒的刷新。
- 1（默认）：表示在执行 commit 时将重做日志缓冲同步写到磁盘，即伴有 fsync 的调用。
- 2：表示将重做日志异步写到磁盘，即写到文件系统的缓存中。不能完全保证在执行 commit 时肯定会写入重做日志文件，只是有这个动作发生。



为了保证事务的 ACID 的持久性，必须将 innodb_flush_log_at_trx_commit 设置为 1，也就是 **每当事务提交时，必须确保事务都已经写入重做日志文件**。如果设置为 0 或 2，都可能发生恢复时部分事务的丢失。

## 表

### 索引组织表

在 InnoDB 存储引擎中，表都是根据主键顺序组织存放的，这种存储方式的表称为 索引组织表（index organized table）。在 InnoDB 存储引擎表中，每张表都有个主键（Primary Key）,r如果在创建表时没有显式地定义主键，则 InnoDB 存储引擎会按如下方式选择或创建主键：

- 首先判断表中是否有**非空**的**唯一索引**（Unique NOT NULL）,如果有，则该列即为主键。
- 如果不符合以上条件，InnoDB 存储引擎自动创建一个 6 字节大小的指针。

当表中有多个非空唯一索引时，InnoDB 存储引擎将选择建表时**第一个定义的非空唯一索引为主键**。注意：主键选择根据的是定义索引的顺序。

可通过如下 SQL 判断表的主键值， _rowid 可以显示表的主键， 但 _rowid 只能用于查看单个列为主键的情况。

```mysql
select sfzhm, _rowid from cyzd.user_info;
```



### InnoDB 逻辑存储结构

InnoDB 存储引擎中，所有数据都被逻辑地存放在一个空间中，称之为表空间（tablespace）。表空间又由段（segment）、区（extent）、页（page）组成。

![InnoDB 逻辑存储结构](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/InnoDB%20%E9%80%BB%E8%BE%91%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.png)

#### 表空间

表空间可以看做是 InnoDB 存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。在默认情况下 InnoDB 存储引擎有一个共享表空间 ibdata1，即所有的数据都存放在这个表空间内。如果设置了参数 innodb_file_per_table（默认为 ON），则每张表内的数据可以单独放到一个表空间内。

如果启用了 innodb_file_per_table 参数，需要注意的是每张表的表空间内存放的只是数据、索引和插入缓冲 Bitmap 页，其他类的数据，如回滚（undo）信息、插入缓冲索引页、系统事务信息、二次写缓冲（double write buffer）等还是存放在原来的共享表空间内。

#### 段

表空间是由各个段组成的，常见的有数据段、索引段、回滚段等。对于 InnoDB ，数据段即为 B+ 树的叶子节点（Leaf node segment），索引段即为 B+ 树的非叶子节点(Non-Leaf node segment)。在 InnoDB 存储引擎中，对段的管理是由引擎自身完成，DBA 不能也没有必要对其进行控制。

#### 区

区是连续页组成的空间，在任何情况下每个区的大小都为 1MB。为了保证区中页的连续性，InnoDB 存储引擎一次从磁盘申请 4~5 个区。在默认情况下，InnoDB 存储引擎页的大小为 16KB，即一个区中一共有 64 个连续的页。

可通过 `innodb_page_size` 改变默认页的大小。

#### 页

页是 InnoDB 磁盘管理的最小单位。在 InnoDB存储引擎中，默认每个页的大小为 16 KB。也可通过 `innodb_page_size` 将页的大小设置为 4K、8K。

在 InnoDB 存储引擎中，常见的页类型有：

- 数据页（B-tree Node）
- undo 页（undo Log Page）
- 系统页（System Page）
- 事务数据页（Transaction system Page）
- 插入缓冲位图页（Insert Buffer Bitmap）
- 插入缓冲空闲列表页（Insert Buffer Free List）
- 未压缩的二进制大对象页（Uncompressed BLOB Page）
- 压缩的二进制大对象页（compressed BLOB Page）



#### 行

InnoDB 存储引擎是面向列的，也就是说数据是按行进行存放的。

InnoDB 存储引擎提供了: `REDUNDANT`, `COMPACT`, `DYNAMIC`, 和 `COMPRESSED` 格式存储行记录数据。

### 数据完整性

- 实体完整性：保证表中所有的行唯一。
- 域完整性：保证数据每列的值满足特定的条件，是否允许为空值等。
- 参照完整性：保证主关键字（被引用表）和外部关键字（引用表）之间的参照关系。即保证两张表之间的关系。







## 参数汇总

| 参数名                        | 含义                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| innodb_change_buffering       | 用来开启 Change Buffer 的各种选项，可选值：inserts、deletes、purges、changes、all（默认）、none。 |
| innodb_change_buffer_max_size | 控制 Change Buffer 最大使用内存的数量，默认 25，最大有效值为 50。 |
| innodb_use_native_aio         | 是否启用 Native AIO，Linux 下默认为 ON                       |
| innodb_flush_neighbors        | 是否开启刷新邻接页特性，建议机械硬盘下开启，固态硬盘下关闭。 |

