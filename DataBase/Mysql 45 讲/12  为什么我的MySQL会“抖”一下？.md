

## 12 | 为什么我的MySQL会“抖”一下？

一条 SQL 语句，正常执行的时候特别快，但是有时也不知道怎么回事，它就会变得特别慢，并且这样的场景很难复现，它不只随机，而且持续时间还很短。看上去，这就像是数据库“抖”了一下。

这个时候，MySQL 可能是在刷脏页（flush）。





### 引发数据库 flush 的情况

1、InnoDB redo log 写满了。这时候系统会停止所有的更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写。

2、系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是 “脏页”，就要先将脏页写到磁盘。

3、MySQL 认为系统空闲的时候。

4、MySQL 正常关闭。MySQL 会把内存的脏页都 flush 到磁盘上。





刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

1、一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；

2、日志写满，更新全部堵住，写性能跌为 0，这种情况对敏感业务来说，是不能接受的。



### InnoDB 刷脏页的控制策略

1、参数`innodb_io_capacity`: InooDB 根据这值判断磁盘能力，建议你设置成磁盘的 IOPS。

> innodb_io_capacity 的默认值为 200，所以如果主机磁盘用的是 SSD，建议改为大，如 20000.



如果设置的过低，InnoDB 认为系统的能力差， 刷脏页刷得特别慢，甚至比脏页生成的速度还慢，这样就造成了脏页累积，影响了查询和更新性能。



2、参数 `innodb_max_dirty_pages_pct` 是脏页比例上限，默认值是 90%。

脏页比例是通过 Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total 得到的。

```sql
select VARIABLE_VALUE into @a from performance_schema.global_status  where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from  performance_schema.global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```



3、参数 `innodb_flush_neighbors` 控制是否刷邻居的脏页，默认值为 0。

即：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延，也就是对于每个邻居数据页，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷。

这个参数在机械盘时代比较有用，可以减少很对随机 IO。

对于 SSD 这类 IOPS 比较高的设备，建议设置为 0，因为这时候 IOPS 往往不是瓶颈。