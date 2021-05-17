### 一、Redis数据库

Redis默认会有16个数据库，且数据库与数据库之间的数据是**隔离**的。

Redis服务器用RedisServer结构体来表示。

```c
struct redisServer{  

    //redisDb数组,表示服务器中所有的数据库
    redisDb *db;  

    //服务器中数据库的数量，可以配置
    int dbnum;  

}; 
```

Redis客户端通过redisClient结构体表示。

```c
typedef struct redisClient{  

    //客户端当前所选数据库
    redisDb *db;  

}redisClient;
```



<iframe id="embed_dom" name="embed_dom" frameborder="0" style="display:block;width:525px; height:245px;" src="https://www.processon.com/embed/603c69261e08530bb1af82d5"></iframe>

​																RedisServer与RedisClient

Redis对每个数据库采用redisDb结构体来表示。

```c
typedef struct redisDb { 
    int id;         // 数据库ID标识
    dict *dict;     // 键空间，存放着所有的键值对              
    dict *expires;  // 过期哈希表，保存着键的过期时间                          
    dict *watched_keys; // 被watch命令监控的key和相应client    
    long long avg_ttl;  // 数据库内所有键的平均TTL（生存时间）     
} redisDb;
```

> Redis的数据库就是使用字典(哈希表)来作为底层实现的，对**数据库的增删改查都是构建在字典(哈希表)的操作之上的**。

> 一般我们将存储所有键值对的`dict`称为**键空间**。

#### 过期策略

过期键是保存在hash表（dict *expires）中，当这些键过期时，redis会做什么呢？

##### 过期策略的分类

- 定时删除：到时间点就把所有过期的键删除  
  - 对内存友好，对CPU不友好
- 惰性删除：每次从键空间取键的时候，判断一下该键是否过期了，如果过期了就删除
  - 对CPU极其友好，对内存不友好
- 定期删除：每隔一段时间去删除过期键，限制删除的执行时长和频率
  - 折中

Redis采用的是**惰性删除+ 定期删除**两种策略。

#### 内存淘汰机制

如果定期删除漏掉了很多过期key，也没及时去查(没走惰性删除)，大量过期key堆积在内存里，导致redis内存块耗尽了，咋整？

我们可以设置内存最大使用量，当内存使用量超出时，会施行**数据淘汰策略**。

| 策略            | 描述                                                 |
| --------------- | ---------------------------------------------------- |
| volatile-lru    | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰 |
| volatile-tll    | 从已设置过期时间的数据集中挑选将要过期的数据淘汰     |
| volatile-random | 从已设置过期时间的数据集中挑选任意数据进行淘汰       |
| allkeys-lru     | 从所有数据集中挑选最近最少使用的数据淘汰             |
| allkeys-random  | 从所有数据集中挑选任意数据进行淘汰                   |
| noeviction      | 当内存使用超过配置的时候会返回错误，不会驱逐任何键   |
| volatile-lfu    | 从已设置过期时间的数据集中挑选使用频率最少的数据淘汰 |
| allkeys-lfu     | 从所有数据集中挑选使用频率最少的数据淘汰             |



> 使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是**热点数据**。可以将内存最大使用量设置为热点数据占用的内存量，然后启用allkeys-lru淘汰策略，将最近最少使用的数据淘汰



### 二、Redis持久化

Redis 提供了两种持久化方式。

- RDB（基于快照）：将某一时刻的所有数据保存到一个RDB文件中。
- AOF(append-only-file)，当Redis服务器执行**写命令**的时候，将执行的**写命令**保存到AOF文件中。



#### RDB（快照持久化）

RDB持久化可以**手动**执行，也可以根据服务器配置**定期**执行。RDB持久化所生成的RDB文件是一个经过**压缩**的**二进制**文件，Redis可以通过这个文件**还原**数据库的数据。

Redis 使用 操作系统的多进程 `COW ( Copy on Write )` 机制来实现快照持久化



##### 原理



![BGSAVE](http://assets.processon.com/chart_image/603d780007912913b4f4550b.png)



#####  手动持久化

有两个命令可以生成RDB文件：

- `SAVE`会**阻塞**Redis服务器进程，服务器不能接收任何请求，直到RDB文件创建完毕为止。（不建议使用）
- `BGSAVE`创建出一个**子进程**，由子进程来负责创建RDB文件，服务器进程可以继续接收请求。

Redis服务器在启动的时候，如果发现有RDB文件，就会**自动**载入RDB文件(不需要人工干预)

- 服务器在载入RDB文件期间，会处于阻塞状态，直到载入工作完成。

##### 配置持久化

```properties
  # 时间策略
save 900 1
save 300 10
save 60 10000

# 文件名称
dbfilename dump.rdb

# 文件保存路径
dir /home/work/app/redis/data/

# 如果持久化出错，主进程是否停止写入
stop-writes-on-bgsave-error yes

# 是否压缩
rdbcompression yes

# 导入时是否检查
rdbchecksum yes
```

save m n, 表示m秒内数据集存在n次修改时，自动触发bgsave。

##### RDB对过期键的策略

- 执行`SAVE`或者`BGSAVE`命令创建出的RDB文件，程序会对数据库中的过期键检查，**已过期的键不会保存在RDB文件中**。
- 载入RDB文件时，程序同样会对RDB文件中的键进行检查，**过期的键会被忽略**。



##### 优劣势

①、优势

（1）RDB文件紧凑，全量备份，非常适合用于进行备份和灾难恢复。

（2）生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作。

（3）RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

②、劣势

RDB快照是一次全量备份，存储的是内存数据的二进制序列化形式，存储上非常紧凑。当进行快照持久化时，会开启一个子进程专门负责快照持久化，子进程会拥有父进程的内存数据，父进程修改内存子进程不会反应出来，所以在快照持久化期间修改的数据不会被保存，可能丢失数据。



#### AOF（文件追加）

AOF 以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。



##### 相关配置

```properties
# 是否开启aof
appendonly yes

# 文件名称
appendfilename "appendonly.aof"

# 同步方式
appendfsync everysec
# appendfsync always     # 每次有数据修改发生时都会写入AOF文件。
# appendfsync everysec   # 每秒钟同步一次，该策略为AOF的默认策略。
# appendfsync no         # 从不同步。高效但是数据不会被持久化。

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 加载aof时如果有错如何处理
aof-load-truncated yes

# 文件重写策略
aof-rewrite-incremental-fsync yes

```



`appendfsync everysec` 它其实有三种模式:

- always：把每个写命令都立即同步到aof，很慢，但是很安全
- everysec：每秒同步一次，是折中方案
- no：redis不处理交给OS来处理，非常快，但是也最不安全

一般情况下都采用 **everysec** 配置，这样可以兼顾速度与安全，最多损失1s的数据。

`aof-load-truncated yes` 如果该配置启用，在加载时发现aof尾部不正确时，会向客户端写入一个log，但是会继续执行，如果设置为 `no` ，发现错误就会停止，必须修复后才能重新加载。

`auto-aof-rewrite-min-size`：执行AOF重写时，文件的最小体积，默认值为64MB。
		`auto-aof-rewrite-percentage`：执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。

只有当`auto-aof-rewrite-min-size`和`auto-aof-rewrite-percentage`两个参数同时满足时，才会自动触发AOF重写，即bgrewriteaof操作。

AOF持久化功能的实现可以分为3个步骤：

- 命令传播：Redis 将执行完的命令、命令的参数、命令的参数个数等信息发送到 AOF 程序中。
- 缓存追加：AOF 程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的 AOF 缓存中。
- 文件写入和保存：AOF 缓存中的内容被写入到 AOF 文件末尾，如果设定的 AOF 保存条件被满足的话， fsync 函数或者 fdatasync 函数会被调用，将写入的内容真正地保存到磁盘中。



##### AOF重写

AOF 文件通过同步 Redis 服务器所执行的命令， 从而实现了数据库状态的记录， 但是， 这种同步方式会造成一个问题： 随着运行时间的流逝， AOF 文件会变得越来越大。另一方面， 有些被频繁操作的键， 对它们所调用的命令可能有成百上千、甚至上万条， 如果这样被频繁操作的键有很多的话， AOF 文件的体积就会急速膨胀， 对 Redis 、甚至整个系统的造成影响。

为了解决以上的问题， Redis 需要对 AOF 文件进行重写（rewrite）： 创建一个新的 AOF 文件来代替原有的 AOF 文件， 新 AOF 文件和原有 AOF 文件保存的数据库状态完全一样， 但新 AOF 文件的体积小于等于原有 AOF 文件的体积。

AOF重写由Redis自行触发(参数配置)，也可以用`BGREWRITEAOF`命令**手动触发**重写操作。

- 要值得说明的是：**AOF重写不需要对现有的AOF文件进行任何的读取、分析。AOF重写是通过读取服务器当前数据库的数据来实现的**！

##### AOF后台重写

Redis将AOF重写程序放到**子进程**里执行(`BGREWRITEAOF`命令)，像`BGSAVE`命令一样fork出一个子进程来完成重写AOF的操作，从而不会影响到主进程。

AOF后台重写是不会阻塞主进程接收请求的，新的写命令请求可能会导致**当前数据库和重写后的AOF文件的数据不一致**！

为了解决数据不一致的问题，Redis服务器设置了一个**AOF重写缓冲区**，这个缓存区会在服务器**创建出子进程之后使用**。 Redis 主进程在接到新的写命令之后， 除了会将这个写命令的协议内容追加到现有的 AOF 文件之外， 还会追加到这个缓存中：

![AOF重写](http://assets.processon.com/chart_image/603d6cde6376896973c97a4b.png)

1、在重写期间，由于主进程依然在响应命令，为了保证最终备份的完整性；因此它依然会写入旧的AOF file中，如果重写失败，能够保证数据不丢失。

2、为了把重写期间响应的写入信息也写入到新的文件中，因此也会为子进程保留一个buf，防止新写的file丢失数据。

3、**重写是直接把当前内存的数据生成对应命令，并不需要读取老的AOF文件进行分析、命令合并**。

4、AOF文件直接采用的**文本协议**，主要是兼容性好、追加方便、可读性高。



> 不管是RDB还是AOF都是先写入一个临时文件，然后通过 `rename` 完成文件的替换工作。

##### AOF对过期键的策略

- 如果数据库的键已过期，但还没被惰性/定期删除，AOF文件不会因为这个过期键产生任何影响(也就说会保留)，当过期的键被删除了以后，会追加一条DEL命令来显示记录该键被删除了
- 重写AOF文件时，程序会对AOF文件中的键进行检查，**过期的键会被忽略**。

##### 优劣势

AOF的优点：丢失数据少(默认配置只丢失一秒的数据)。

AOF的缺点：恢复数据相对较慢，文件体积大

如果Redis服务器**同时开启**了RDB和AOF持久化，服务器会**优先使用AOF文件**来还原数据(因为AOF更新频率比RDB更新频率要高，还原的数据更完善)



#### 混合持久化

快照和aof日志都有各自的痛点

- 快照因为是每隔一段时间持久化一次, 就会丢失宕机时刻与上一次持久化之间的数据
- aof 因为存储的是指令序列, 恢复重放时要花费很长时间

Redis 4.0 提供了更好的混合持久化选项

- 将 rdb 文件的内容和增量的 aof 日志放在一起
- aof 日志只存储 rdb 持久化开始到当前发生的增量日志
- 重启时, 先加载 rbd 内容, 再重放增量 aof 日志

### 参考资料

[1] https://mp.weixin.qq.com/s/lxXxsRL4GK2bSFkAqeLurA

[2] https://zhuanlan.zhihu.com/p/105587132

[3] https://juejin.cn/post/6844903655527677960

[4] http://www.jwsblog.com/archives/70.html

[5] https://redisbook.readthedocs.io/en/latest/internal/aof.html

[6] RESP:https://www.cnblogs.com/tommy-huang/p/6051577.html