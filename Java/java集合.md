[TOC]



# Collection

### 集合和数组的区别

- 1、长度的区别
  - 数组的长度固定
  - 集合的**长度可变**
- 2、元素的数据类型
  - 数组可以存储基本数据类型，也可以存储引用类型
  - 集合**只能存储引用类型**，基本类型会自动装箱

### Collection的结构体系

![collection](http://image.yzqfrist.com/QQ%E6%88%AA%E5%9B%BE20210507074015.png)

### Collection的基础功能

![image-20210507075323746](http://image.yzqfrist.com/image-20210507075323746.png)



# List集合

## 一、ArrayList解析

![image-20210506070109745](http://image.yzqfrist.com/image-20210506070109745.png)

> ArrayList的实现的非同步的，对结构性修改（对元素删除或者新增或者改变size， 修改元素的值不算是结构性修改）必须进行同步。
>
> ```java
> List list = Collections.synchronizedList(new ArrayList(...));
> ```
>
> ArrayList类返回迭代器和 listIterator方法返回的迭代器是**快速失败（fast-fail）**的，如果在创建迭代器之后的任何时间以任何方式对列表进行了结构性修改，除了迭代器本身的remove和add方法外，迭代器会**尽最大努力**抛出ConcurrentModificationException。

![ArrayList01](http://image.yzqfrist.com/ArrayList01.png)

### 1.1 Add方法

#### 1.1.1 add(E e) ：直接添加元素

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

基本实现：

- 首先检查一下数组的容量是否足够
  - 足够：直接添加
  - 不足够：扩容
    - **扩容到原来的1.5倍**
    - 第一次扩容之后，如果容量还是小于minCapacity，则将容量扩容至minCapacity，调用**copyOf**拷贝元素
    - 添加元素

![ArrayList02](http://image.yzqfrist.com/ArrayList02.png)

进入看copyOf方法

![image-20210506195822788](http://image.yzqfrist.com/image-20210506195822788.png)

#### 1.1.2 add(int index, E element)：添加元素到指定位置

![ArrayList03](http://image.yzqfrist.com/ArrayList03.png)

### 1.2 get方法

- 检查角标
- 返回元素

![image-20210506200052007](http://image.yzqfrist.com/image-20210506200052007.png)

```java
//检查角标
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
//返回元素
E elementData(int index) {
    return (E) elementData[index];
}
```

### 1.3 set方法

```java
public E set(int index, E element) {
	rangeCheck(index); //检查角标

	E oldValue = elementData(index);//获取旧值
	elementData[index] = element; //赋新值
	return oldValue; //返回旧值
}
```

### 1.4 remove方法

移除指定位置处的值，并将该位置右边的值左移。

```java
public E remove(int index) {
    rangeCheck(index);//检查角标

    modCount++; //结构性修改
    E oldValue = elementData(index);//获取旧值

    int numMoved = size - index - 1;//需要左移的元素个数
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;//返回旧值
}
```

### 1.5 trimToSize方法

将ArrayList实例的容量削减为列表的当前大小。

![image-20210506201826766](http://image.yzqfrist.com/image-20210506201826766.png)

### 1.6 总结说明

- ArrayList是基于**动态数组实现的，在增删的时候，需要数组的拷贝复制**。(navite ⽅法由C/C++实现)
- ArrayList的**默认初始容量为10**，每次扩容，先增加**原先容量的一半，即扩容为1.5倍**。
- 删除元素时不会减少容量，若希望减少容量，则调用trimToSize方法。
- 非线程安全，可以存放null值。

## 二、LinkedList解析

![image-20210506231434811](http://image.yzqfrist.com/image-20210506231434811.png)



>LinkedList的实现的非同步的，对结构性修改（对元素删除或者新增或者改变size， 修改元素的值不算是结构性修改）必须进行同步。
>
>```java
> List list = Collections.synchronizedList(new LinkedList(...));
>```
>
>LinkedList类返回迭代器和 listIterator方法返回的迭代器是**快速失败（fast-fail）**的，如果在创建迭代器之后的任何时间以任何方式对列表进行了结构性修改，除了迭代器本身的remove和add方法外，迭代器会**尽最大努力**抛出ConcurrentModificationException。

![image-20210506233601160](http://image.yzqfrist.com/image-20210506233601160.png)



```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```



### 2.1 add方法

向链表的最后添加元素。

```java

public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

### 2.2 remove方法

双向链表中删除元素。

```java
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}


E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

### 2.4 get方法

```java
public E get(int index) {
    checkElementIndex(index);//检查index的合法性 index >= 0 && index < size
    return node(index).item; //返回特定位置处的元素的值
}

//双端链表 从最近的一端开始获取元素
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) { //如果index小于size的一半， 从头部遍历获取
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else { //否则，从尾部遍历获取
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```



### 2.5 set方法

```java
public E set(int index, E element) {
    checkElementIndex(index);//检查index的合法性
    Node<E> x = node(index);//获取指定位置处的node
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

### 2.6 总结说明

- LinkedList是由**双端链表**实现的，非同步的。



# Map集合

## 一、散列表

散列表为每个对象计算出一个整数，称为**散列码**。根据这些计算出来的整数（散列码）保存在对应的位置上。

在Java中，散列表用的是**链表数组**实现的，每个列表称之为桶。

一个桶上可能会遇到被占用的情况（hashcode散列码相同，就存储在同一个位置上），这种情况是无法避免的的，称之为：**散列冲突**。

- 此时需要用该对象与桶上的对象进行比较，看看该对象是否存在桶上，不存在则添加之。
- 在**JDK1.8**中,**桶满时会从链表变成平衡二叉树**。

如果散列表太满，是需要对**散列表再散列**，创建一个桶数更多的散列表，并将原有的元素插入到新表中，丢弃原来的表。

- 装载因子（load factor）**决定**了何时对散列表再散列。
- 装载因子**默认为0.75**，如果表中超过75%的位置填入了元素，那么这个表就会用**双倍**的桶数自动进行再散列。

## 二、红黑树

- 红黑树是二叉搜索树。
- 根节点是黑色。
- 每个叶子节点都是黑色的空节点（NIL节点）。
- 每个红色节点的两个子节点都是黑色。（从每个叶子到根的所有路径上不能有两个连续的红色节点）
- 从任一节点到其每个叶子的所有路径都包含相同相同数目的黑色节点（每一条树链上的黑色节点数量必须相等）

## 三、HashMap解析

从HashMap的头部注释中可以得到一些信息：

- 允许键值为null
- 除了非同步和允许为null，和hashtable没啥区别
- 不保证顺序

- 初始容量太高和装载因子太低都不好
- 当初始容量 * 装载因子小于哈希表填充量时，哈希表会进行再散列，桶数会*2
- 非同步，若要使用同步可以如下

```java
 Map m = Collections.synchronizedMap(new HashMap(...));
```

- 快速失败的迭代器

类继承图：

![image-20210507201610204](http://image.yzqfrist.com/image-20210507201610204.png)

相关重要属性：

```java

//初始容量16，必须为2的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//最大容量为2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认装载因子为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//当添加一个元素到至少有TREEIFY_THRESHOLD的节点的桶中，桶中链表将被转化为树形结构
static final int TREEIFY_THRESHOLD = 8;

//同上，当桶中节点数量小于等于6时，将树形结构转化为链表
static final int UNTREEIFY_THRESHOLD = 6;

//桶可能被转化为树形结构的最小容量
static final int MIN_TREEIFY_CAPACITY = 64;
```

内部Node类：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

### 3.1 构造方法

HashMap有4个构造方法

![image-20210507204127326](http://image.yzqfrist.com/image-20210507204127326.png)

```java
public HashMap(int initialCapacity, float loadFactor) {
	//判断输入的大小是否合理
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
    
    this.loadFactor = loadFactor;//初始化装载因子
    this.threshold = tableSizeFor(initialCapacity);
}

//Returns a power of two size for the given target capacity.
//返回一个大于输入参数且最接近2的整数次幂的数
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

![image-20210507205900495](http://image.yzqfrist.com/image-20210507205900495.png)



第一次初始化散列表：threshold被初始化为 容量大小 * loadFactor。

![image-20210507215306412](http://image.yzqfrist.com/image-20210507215306412.png)

### 3.2 put方法

```java
//创建键值的映射，如果键存在，则替换键对应的值
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
//计算Hash值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可见，哈希值的计算时**先得到key的hashCode值，然后与key的hashCode值得高16位做异或运算**得到。为什么要这样做呢？

![image-20210507211026864](http://image.yzqfrist.com/image-20210507211026864.png)

我们是根据key的哈希值来保存到散列表中的，默认初始容量为16，也就是0-15的位置，即 `tab[i = (n - 1) & hash]`。此时，在做`&`运算的时候，仅仅后**4位有效**，那如果key的哈希值高位变化大，低位变化小，会导致计算出来的Hash值相同的很多。

所以，设计者**将key的哈希值的高位也做了运算（与高16位做异或运算，使得在做`&`运算时，此时的低位实际上是高位与低位的结合），这样增加了随机性**，减少了碰撞冲突的可能性。

![HashMap01](http://image.yzqfrist.com/HashMap01.png)

### 3.3 get方法

![HashMap02](http://image.yzqfrist.com/HashMap02.png)

### 3.4 remove方法

![HashMap03](http://image.yzqfrist.com/HashMap03.png)

### 3.5 总结说明

- JDK1.8 HashMap的底层是：**数组 + 链表（散列表）+ 红黑树*
- 当**装载因子0.75 * 初始容量16**小于散列表元素时，散列表再散列为**2倍**。
- HashMap的hash值计算 `(h = key.hashCode()) ^ (h >>> 16)`，将key的哈希值与key哈希值的高16位进行异或操作，使得我们将元素放入哈希表的时候**增加了一定的随机性**。
- 值得注意的是：**并不是桶子上有8位元素的时候它就能变成红黑树，它得同时满足散列表容量大于64才行**。

## 四、LinkedHashMap解析

从类的顶部注释可了解到：

- 迭代有序，散列表+ 双向链表实现。

- 装载因子和初始容量会影响性能。
- 非同步，允许为null
- 链表的访问顺序：`insertion-ordered` 和` access-ordered`(适合创建LRU队列)。**在`insertion-ordered`条件下，修改已有key的value，不是结构性修改。`access-ordered`条件下，使用get方法已经是结构性修改了**。
- 迭代与容量无关：迭代的是链表，只和数据有关。而HashMap底层是数组，所以与 容量有关。

类继承图：

![image-20210509145035700](http://image.yzqfrist.com/image-20210509145035700.png)

可知LinkedHashMap继承自HashMap，其装载因子和初始容量都继承自HashMap。

相关重要类属性：

![image-20210509145443020](http://image.yzqfrist.com/image-20210509145443020.png)

### 4.1 构造方法

LinkedHashMap有5个构造方法：

![image-20210509151120703](http://image.yzqfrist.com/image-20210509151120703.png)

```java
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor); //调用HashMap的构造方法
    this.accessOrder = accessOrder; //访问方式： true - access-order  false - insertion-order
}
```

其他构造方法的`accessOrder`皆为`false`, 即**默认为插入顺序**。

### 4.2 put方法

LinkedHashMap的`put`方法和父类HashMap的`put`方法是同一个，不过`put`方法中调用的创建节点的`newNode`方法还是LinkedHashMap的重写方法。

### 4.3 get方法

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null) //getNode方法是HashMap的
        return null;
    if (accessOrder) //如果是访问顺序 
        afterNodeAccess(e); //把该节点放到最后面 然后返回值
    return e.value;
}
```

### 4.4 remove方法

LinkedHashMap的`remove`方法和父类HashMap的`remove`方法是同一个，不过`remove`方法中调用的`afterNodeRemoval`方法还是LinkedHashMap的重写方法。

### 4.5 总结说明

- LinkedHashMap可以设置两种遍历顺序：
  - 访问顺序（acces-ordered）：
  - 插入顺序（insertion-ordered） 默认

对于访问顺序，它是LRU(最近最少使⽤)算法的实现，要使⽤它要么重写LinkedListMap的⼏个⽅法 ( `removeEldestEntry(Map.Entry eldest) `和 `afterNodeInsertion(boolean evict)` )，要 么是扩展成LRUMap来使⽤，不然设置为访问顺序（access-ordered）的⽤处不⼤

## 五、TreeMap解析

从类顶部的注释可以了解到：

- 底层是**红黑树**，可以根据key自然排序，也可在构造方法中传递 Comparator实现map的排序
-  `containsKey`, `get`, `put` 和 `remove` 的时间复杂度为log(n)

- 非同步
- 使⽤Comparator或者Comparable来⽐较key是否相等与排序
- TreeMap实现了NavigableMap接⼝，⽽NavigableMap接⼝继承着SortedMap接⼝，致使 TreeMap是有序的！

类继承图

![image-20210512064912229](http://image.yzqfrist.com/image-20210512064912229.png)

相关重要属性：

```java
private final Comparator<? super K> comparator;//用于维护树的映射顺序的比较器，如果为null，则为自然排序

private transient Entry<K,V> root;//红黑树的根节点

private transient int size = 0;//红黑树中节点数量

private transient int modCount = 0;//结构性修改的次数
```

### 5.1 构造方法

TreeMap有4个构造方法：

![image-20210512065338909](http://image.yzqfrist.com/image-20210512065338909.png)

构造方法大多与comparator有关：

![image-20210512065735676](http://image.yzqfrist.com/image-20210512065735676.png)

### 5.2 put方法

![img](http://image.yzqfrist.com/162b908ce85f2498)

下面是`compare(Object k1, Object k2)`方法吗，如果设置key为null，会抛出异常。

![image-20210512123828297](http://image.yzqfrist.com/image-20210512123828297.png)

### 5.3 get方法

- 调用`getEntry`方法
  - 若comparator不为null， 调用 `getEntryUsingComparator`，内部又调用`compare`方法
  - 否则，调用`Comparable`的`compareTo`方法

```java
public V get(Object key) {
    Entry<K,V> p = getEntry(key); //找到返回value 找不到返回null
    return (p==null ? null : p.value);
}


final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}

final Entry<K,V> getEntryUsingComparator(Object key) {
    @SuppressWarnings("unchecked")
    K k = (K) key;
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = cpr.compare(k, p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
    }
    return null;
}
```

### 5.4 remove方法

首先获取到该Entry，如果为空，则返回null，否则调用`deleteEntry`进行删除，并返回删除的节点值。

```java
public V remove(Object key) {
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
```

`deleteEntry`方法主要就是**删除节点并平衡红黑树**。

### 5.5 总结说明

- 由于底层是红⿊树，那么时间复杂度可以保证为log(n) 
- key不能为null，为null为抛出NullPointException的 
- 想要⾃定义⽐较，在构造⽅法中传⼊Comparator对象，否则使⽤key的⾃然排序来进⾏⽐较 
- TreeMap⾮同步的，想要同步可以使⽤Collections来进⾏封装

## 六、ConcurrentHashMap

- 底层是散列表+ 红黑树，和HashMap一样。
- 支持高并发的检索和更新，即线程安全且检索操作不用加锁
- 检索操作（包括get）是非阻塞的，检索出来的值是最新设置的值。
- 迭代器同一时间只能给一个线程使用。

- 包括 size，isEmpty和containsValue等统计的方法，最好只在单线程情况下使用，在多线程情况下返回的值不是精确的。
- 当冲突较多时，会自动扩容，不过扩容时一个非常耗费资源的操作，最好在构造方法中就初始化好要放置的元素个数和装载因子。

- key和value 不允许为null

重要的域：

![ConcurrentHashMap field](http://image.yzqfrist.com/ConcurrentHashMap%20field.png)

### 6.1 构造方法

![image-20210513120439837](http://image.yzqfrist.com/image-20210513120439837.png)



![image-20210513120622743](http://image.yzqfrist.com/image-20210513120622743.png)

### 6.2 put方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```



![image-20210513123831067](http://image.yzqfrist.com/image-20210513123831067.png)



![image-20210513123949222](http://image.yzqfrist.com/image-20210513123949222.png)

再看`initTable`方法：

![image-20210513124242080](http://image.yzqfrist.com/image-20210513124242080.png)

### 6.3 get方法

从顶部注释可知，get方法**不用加锁，是非阻塞的**。

可以发现，Node节点是重写的，设置了`volatile`关键字修饰，致使它每次获取的都是**最新**设置的值。

![image-20210513210956311](http://image.yzqfrist.com/image-20210513210956311.png)

### 6.4 总结说明

- 底层结构是散列表（数组 + 链表）+ 红黑树
- HashTable是将所有的方法进行同步，效率低下。而ConcurrentHashMap作为一个高并发的容器，它是通过**部分锁定+CAS算法**来进行实现线程安全的。
- 在高并发环境下，统计数据（计算size.. 等等）其实是无意义的，因为下一时刻size值就变化了。
- get方法是非阻塞的，无锁的。原理是重写Node类，通过volatile修饰，来实现每次获取的都是**最新**设置的值。
- **ConcurrentHashMap的key和value都不能为null**。



# Set

## 一、HashSet

类继承图

![image-20210514123027388](http://image.yzqfrist.com/image-20210514123027388.png)

从顶部注释可知：

- 底层实际上是一个HashMap实例，不保证迭代有序
- 实现set接口，允许为null
- 如果很看重性能，不要将初始容量设置得太高或者负载因子太低
- 非同步

![image-20210514123450438](http://image.yzqfrist.com/image-20210514123450438.png)

## 二、TreeSet

类继承图

![image-20210514123558906](http://image.yzqfrist.com/image-20210514123558906.png)

从顶部注释可知：

- 底层是一个TreeMap实例，实现了NavigableSet接口
- 根据Comparable实现自然排序，或者在构造方法传递Comparator对象实现排序
- 时间复杂度保证log（n）
- 非同步

![image-20210514124011036](http://image.yzqfrist.com/image-20210514124011036.png)

## 三、LinkedHashSet

类继承图

![image-20210514124137384](http://image.yzqfrist.com/image-20210514124137384.png)

从顶部注释可知：

- 迭代有序
- 允许为null
- **底层实际上是一个HashMap + 双向链表实例（其实就是LinkedHashMap）**
- 非同步

- 性能比HashSet差一丢丢，因为要维护一个双向链表
- 初始容量与迭代无关，LinkedHashSet迭代的是双向链表。



## 四、Set 集合总结

| HashSet       | 无序、允许为null，底层是HashMap（散列表+红黑树），非线程同步 |
| ------------- | ------------------------------------------------------------ |
| TreeSet       | 有序，不允许为null，底层是TreeMap（红黑树），非线程同步      |
| LinkedHashMap | 迭代有序，允许为null，底层是HashMap+双向链表，非线程同步     |

# CopyOnWriteArrayList

⼀般来说，我们会认为：CopyOnWriteArrayList是同步List的替代品，CopyOnWriteArraySet是同步 Set的替代品。

- 线程安全的容量，底层是通过复制数组的方式实现的。
- 遍历不会抛出ConcurrentModificationException，且遍历时不用额外加锁
- 元素可以为null
- 不支持迭代器本身的remove、set、add操作，会抛出UnsupportedOperationException。

### 基本结构

```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}

/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}

/**
 * Creates an empty list.
 */
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

可见，CopyOnWriteArrayList底层是数组，加锁交给ReentrantLock。

### 常见方法

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //得到原数组的长度和元素
        Object[] elements = getArray();
        int len = elements.length;
        //复制一个新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //添加元素到新数组
        newElements[len] = e;
        //将volatile Object[] array 指向新数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

final Object[] getArray() {
    return array;
}


public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    //加锁
    lock.lock();
    try {
        //获取到原数组和旧值
        Object[] elements = getArray();
        E oldValue = get(elements, index);
		//如果旧值不和要设置的值相等
        if (oldValue != element) {
            //复制新数组，操作在新数组中完成
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

```

总结： 在修改时，复制出⼀个**新数组**，修改的操作在新数组中完成，最后将新数组交由array变量指向。 **写加锁，读不加锁**



常用的方法实现我们已经基本了解了，但还是不知道为啥能够在容器遍历的时候对其进行修改而不抛出异常。所以，来看一下他的迭代器吧：

### 剖析为什么遍历时不用调用者显式加锁  

```java
 // 1. 返回的迭代器是COWIterator
   public Iterator<E> iterator() {
     return new COWIterator<E>(getArray(), 0);
   }


   // 2. 迭代器的成员属性
   private final Object[] snapshot;
   private int cursor;

   // 3. 迭代器的构造方法
   private COWIterator(Object[] elements, int initialCursor) {
     cursor = initialCursor;
     snapshot = elements;
   }

   // 4. 迭代器的方法...
   public E next() {
     if (! hasNext())
       throw new NoSuchElementException();
     return (E) snapshot[cursor++];
   }

   //.... 可以发现的是，迭代器所有的操作都基于snapshot数组，而snapshot是传递进来的array数组
```


CopyOnWriteArrayList在使用迭代器遍历的时候，操作的都是**原数组**！

### CopyOnWriteArrayList缺点

- **内存占用**：如果CopyOnWriteArrayList经常要增删改里面的数据，经常要执行add()、set()、remove()的话，那是比较耗费内存的。

​	   因为我们知道每次add()、set()、remove()这些增删改操作都要**复制一个数组**出来。

- **数据一致性**：CopyOnWrite容器**只能保证数据的最终一致性，不能保证数据的实时一致性**。



CopyOnWriteArraySet的原理就是CopyOnWriteArrayList。