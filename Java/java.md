##### ThreadLocal

ThreadLocal是一个本地线程副本变量工具类。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现无状态的调用，特别适用于各个线程依赖不同的变量值完成操作的场景。

![img](http://image.yzqfrist.com/img/20210220103729.png)

ThreadLocal的核心机制：

- 每个**Thread线程内部都有一个map（ThreadLocalMap）**。
- Map里面存储线程本地对象（key）和线程的变量副本（value）
- 但是，Thread内部的Map是由ThreadLocal维护的，由ThreadLocal负责向map获取和设置线程的变量值。

ThreadLocalMap是ThreadLocal的内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也独立实现。

在ThreadLocalMap中，也是用Entry来保存K-V结构数据的。但是Entry中key只能是ThreadLocal对象，这点被Entry的构造方法已经限定死了。

Entry继承自WeakReference（弱引用，生命周期只能存活到下次GC前），但只有Key是弱引用类型的，Value并非弱引用。

和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用**线性探测**的方式。

**ThreadLocalMap的key是弱引用，而Value是强引用。**这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

既然Key是弱引用，那么我们要做的事，就是在调用ThreadLocal的get()、set()方法时完成后再调用remove方法，将Entry节点和Map的引用关系移除，这样整个Entry对象在GC Roots分析后就变成不可达了，下次GC的时候就可以被回收。

##### 反射

在 Java API 中，提供了获取 Class 类对象的三种方法：

**第一种**，使用 Class.forName 静态方法。

前提：已明确类的全路径名。  	clazz1 = Class.forName("com.reflection.User");

**第二种**，使用 .class 方法。

说明：仅适合在编译前就已经明确要操作的 Class    Class clazz2 = User.class;

**第三种**，使用类对象的 getClass() 方法。

适合有对象示例的情况下    User user = new User();  Class clazz3 = user.getClass();



​      