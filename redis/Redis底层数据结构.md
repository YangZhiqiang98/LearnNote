### 前言

Redis  支持丰富的数据结构，常用的有string、list、hash、set、sortset。

Redis的存储是以key-value 结构存储的，Redis 中的key一定是字符串，value则是string、list、hash、set、sortset 这些Redis提供的数据结构。

但注意：key/value都是对象，Redis创建键值对时，都会创建两个对象（键对象和值对象）。

Redis中的每个对象都由一个redisObject结构来表示，每个**对象都有type(类型)、encoding（编码）、ptr(指向底层数据结构的指针)**来表示。同一种数据结构，不同编码对应的底层实现是不一样的。

![Redis数据类型与底层数据结构关系](http://assets.processon.com/chart_image/603b55666376896973c595e8.png)

在 Redis 中，常用的 5 种数据类型和应用场景如下：

- **String：** 缓存、计数器、分布式锁等。
- **List：** 链表、队列、微博关注人时间轴列表等。
- **Hash：** 用户信息、Hash 表等。
- **Set：** 去重、赞、踩、共同好友等。
- **Zset：** 访问量排行榜、点击量排行榜等。

### SDS 简单动态字符串

```c
struct sdshdr{

    // 字节数组，用于保存字符串
    char buf[];

    // 记录buf数组中已使用的字节数量，也是字符串的长度
    int len;

    // 记录buf数组未使用的字节数量
    int free;
}
```

![C语言字符与SDS](http://assets.processon.com/chart_image/603b57ec7d9c087bdf6f4f69.png)

#### SDS与C语言字符串的区别

（1）**字符串长度**    C语言 $O(N)$  SDS  $O(1)$

   (2) **内存重新分配**

-  C语言涉及修改字符串的时候会重新分配内存。	

- SDS优化策略
  -   **空间预分配**：SDS 被修改后，程序不仅会为 SDS 分配所需要的必须空间，还会分配额外的未使用空间。分配规则如下：如果对 SDS 修改后，len 的长度小于 1M，那么程序将分配和 len 相同长度的未使用空间。举个例子，如果 len=10，重新分配后，buf 的实际长度会变为 10(已使用空间)+10(额外空间)+1(空字符)=21。如果对 SDS 修改后 len 长度大于 1M，那么程序将分配 1M 的未使用空间。				
  -  **惰性空间释放**：SDS缩短时，并不会回收多余的内存空间，而是使用free空间字段将多出来的空间记录下来，如果后续有变更操作，直接使用free中记录的空间，减少了内存的分配。

（3）**二进制安全**:C语言遇到'\0'会结束，在二进制数据中不安全，SDS根据len长度判断字符串结束，不存在二进制不安全问题。

### LinkedList  双端列表



```c
typedef strcut listNode{

    //前置节点
    strcut listNode  *pre;

    //后置节点
    strcut listNode  *pre;

    //节点的值
    void  *value;

}listNode
    
    
typedef struct list{

    //表头结点
    listNode  *head;

    //表尾节点
    listNode  *tail;

    //链表长度
    unsigned long len;

    //节点值复制函数
    void *(*dup) (viod *ptr);

    //节点值释放函数
    void  (*free) (viod *ptr);

    //节点值对比函数
    int (*match) (void *ptr,void *key);

}list
```



![数据结构LinkedList](http://assets.processon.com/chart_image/603b7dbd7d9c087bdf6f84d3.png)



Redis 的链表实现的特性可以总结如下：

- 双端：链表节点带有 prev 和 next 指针，获取某个节点的前置节点和后置节点的复杂度都是 O（1）。
- 无环：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL，对链表的访问以 NULL 为终点。
- 带表头指针和表尾指针：通过 list 结构的 head 指针和 tail 指针，程序获取链表的表头节点和表尾节点的复杂度为 O（1）。
- 带链表长度计数器：程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数，程序获取链表中节点数量的复杂度为 O（1）。
- **链表节点使用 void* 指针来保存节点值，可以用于保存各种不同类型的值**。

### ZipList 压缩列表

解决双端列表的问题： 双端列表在存储小数据时，要对应的保存前后指针等数据，如果数据较小，浪费空间，同时由于反复的申请和释放也容易导致内存碎片化。

压缩列表(ziplist)是Redis为了节约内存而开发的，是由一系列的**特殊编码的连续内存块**组成的**顺序性**数据结构。

当一个列表只有少量数据的时候，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么 Redis 就会使用压缩列表来做列表键的底层实现。

```c
struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}
```

![压缩列表zipList](http://assets.processon.com/chart_image/603b81906376896973c5d412.png)

> 压缩列表从表尾节点**倒序遍历**，首先指针通过zltail偏移量指向表尾节点，然后通过指向**节点记录的前一个节点的长度依次向前遍历访问整个压缩列表**。



### **quicklist** 

**quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来。**

![qucikList](http://assets.processon.com/chart_image/603b84ab6376896973c5d79e.png)

### 字典（哈希表）

https://mp.weixin.qq.com/s/XkEK8_OM0OdjpIR2-FkeIQ

**Redis中有两个哈希表**：

- ht[0]：用于存放**真实**的`key-vlaue`数据
- ht[1]：用于**扩容(rehash)**

在对哈希表进行扩展或者收缩操作时，reash过程并不是一次性地完成的，而是**渐进式**地完成的。Java一次性直接rehash

Redis中哈希算法和哈希冲突跟Java实现的差不多，它俩**差异**就是：

- Redis哈希冲突时：是将新节点添加在链表的**表头**。
- JDK1.8后，Java在哈希冲突时：是将新的节点添加到链表的**表尾**。

```c
	   typedef struct dictht{

        //哈希表数组
        dictEntry **table;  

        //哈希表大小
        unsigned long size;    

        //哈希表大小掩码，用于计算索引值
        //总是等于size-1
        unsigned long sizemark;     

        //哈希表已有节点数量
        unsigned long used;

   }dictht
```



```c
typedef struct dict {

    //类型特定函数
    dictType *type;

    //私有数据
    void *privdata;

    //哈希表
    dictht ht[2];

    //rehash索引
    //当rehash不进行时，值为-1
    int rehashidx;  

}dict;

```

Redis具体是rehash时这么干的：

- (1:在字典中维持一个索引计数器变量rehashidx，并将设置为0，表示rehash开始。
- (2:在rehash期间每次对字典进行增加、查询、删除和更新操作时，**除了执行指定命令外**；还会将ht[0]中rehashidx索引上的值**rehash到ht[1]**，操作完成后rehashidx+1。
- (3:字典操作不断执行，最终在某个时间点，所有的键值对完成rehash，这时**将rehashidx设置为-1，表示rehash完成**
- (4:在渐进式rehash过程中，字典会同时使用两个哈希表ht[0]和ht[1]，所有的更新、删除、查找操作也会在两个哈希表进行。例如要查找一个键的话，**服务器会优先查找ht[0]，如果不存在，再查找ht[1]**，诸如此类。此外当执行**新增操作**时，新的键值对**一律保存到ht[1]**，不再对ht[0]进行任何操作，以保证ht[0]的键值对数量只减不增，直至变为空表。

### skiplist 跳跃表

sorted set 类型的排序功能便是通过「跳跃列表」数据结构来实现。

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

- [漫画算法：什么是跳跃表?](https://www.jianshu.com/p/dc252b5efca6) 

跳跃表支持平均 O（logN）、最坏 O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

跳表在链表的基础上，增加了多层级索引，通过索引位置的几个跳转，实现数据的快速定位。

![zipList简单原理图](http://assets.processon.com/chart_image/603b89727d9c087bdf6f9252.png)

这是跳跃表的简单原理图，每一层都有一条有序的链表，最底层的链表包含了所有的元素。这样跳跃表就可以支持在 O(logN) 的时间复杂度里查找到对应的节点。

![zipList](http://assets.processon.com/chart_image/603b8adfe0b34d42faa8265b.png)



### 整数数组（intset）

当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。结构如下：

```c
typedef struct intset{
     //编码方式
     uint32_t encoding;
     //集合包含的元素数量
     uint32_t length;
     //保存元素的数组
     int8_t contents[];
}intset;
```



contents 数组是整数集合的底层实现：整数集合的每个元素都是 contents 数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。length 属性记录了整数集合包含的元素数量，也即是 contents 数组的长度。



### **合理的数据编码**

**String**：存储数字的话，采用int类型的编码，如果是非数字的话，采用 raw 编码；

**List**：字符串长度及元素个数小于一定范围使用 ziplist 编码，任意条件不满足，则转化为 linkedlist 编码；

**Hash**：hash 对象保存的键值对内的键和值字符串长度小于一定值及键值对；

**Set**：保存元素为整数及元素个数小于一定范围使用 intset 编码，任意条件不满足，则使用 hashtable 编码；

**Zset**：zset 对象中保存的元素个数小于及成员长度小于一定值使用 ziplist 编码，任意条件不满足，则使用 skiplist 编码。



###  Redis对象一些细节

- (1：服务器在执行某些命令的时候，会**先检查给定的键的类型**能否执行指定的命令。

  比如我们的数据结构是sortset，但你使用了list的命令。这是不对的，服务器会检查一下我们的数据结构是什么才会进一步执行命令

- (2：Redis的对象系统带有**引用计数**实现的**内存回收机制**。

  对象不再被使用的时候，对象所占用的内存会释放掉

- (3：Redis会共享值为0到9999的字符串对象

- (4：对象**会记录自己的最后一次被访问时间**，这个时间可以用于计算对象的空转时间。





### 参考资料

[1] https://mp.weixin.qq.com/s/XkEK8_OM0OdjpIR2-FkeIQ

[2] https://mp.weixin.qq.com/s/pCs2YwysJEnnz0AW_zkXIw

[3] https://mp.weixin.qq.com/s/DFOGihcocb0VGBqDD6zWWA