

### 基础

#### 接口和抽象类的区别

相同点：

(1)都不能被实例化

(2)接口的实现类或抽象类的子类都只有实现了接口或抽象类中的方法后才能实例化。

不同点：

(1)接口只有定义，不能有方法的实现，java 1.8中可以定义default方法体,接口中可以定义静态的非抽象的方法，而抽象类可以有定义与实现，方法可在抽象类中实现

(2)实现接口的关键字为implements，继承抽象类的关键字为extends。一个类可以实现多个接口，但一个类只能继承一个抽象类。所以，使用接口可以间接地实现多重继承。

(3)接口强调特定功能的实现，而抽象类强调所属关系。

(4)接口成员变量默认为public static final，必须赋初值，不能被修改；其所有的成员方法都是public、abstract的。抽象类中成员变量默认default，可在子类中被重新定义，也可被重新赋值；抽象方法被abstract修饰，不能被 private、static、synchronized 和 native 等修饰，必须以分号结尾，不带花括号。

(5)接口被用于常用的功能，便于日后维护和添加删除，而抽象类更倾向于充当公共类的角色，不适用于日后重新对立面的代码修改。功能需要累积时用抽象类，不需要累积时用接口。


> jdk8开始，**接口中可以定义静态的非抽象的方法**，直接使用接口名调用静态方法，但是它的实现类的类名或者实例却不可以调用接口中的静态方法。也可以定义普通的非抽象的方法，普通的非抽象方法要在返回值前加上default，对于普通的非抽象方法必须使用子类的实例来调用。如果有多个接口定义了相同的默认方法，实现多个这些接口时必须重写默认方法，否则编译失败。jdk8的接口中，开始允许使用关键字default。

> 如果有两个接口中的静态方法一模一样，并且一个实现类同时实现了这两个接口，此时并不会产生错误，因为JDK8只能通过接口类调用接口中的静态方法，所以对编译器来说是可以区分的。
>
> 但是如果两个接口中定义了一模一样的默认方法，并且一个实现类同时实现了这两个接口,那么会产生默认方法冲突，编译器会提示接口冲突，让我们自行解决，那么必须在实现类中**重写默认方法**，否则编译失败。 
>
> 当一个接口实现另一个接口时，子接口与父接口存在相同的默认方法时，将采用**父类接口的默认方法**。
>
> 当一个类既继承父类又实现接口时，如果父类中的方法与接口中的默认方法冲突，将会执行父类中的方法，而不是接口中的方法，这种情况被称为**类优先选择**，方便兼容JDK8之前的版本。

##### 相关文章

[1] [java接口和抽象类区别是什么_Java接口和抽象类有什么区别_Problem Solver的博客-CSDN博客](https://blog.csdn.net/weixin_42715262/article/details/114352431)

[2] [JDK8中接口的变化_不学到秃头不改名的博客-CSDN博客](https://blog.csdn.net/qq_42107430/article/details/105422554)



#### 异常

![image-20220213110834341](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220213110837.png)

从继承关系可知：`Throwable`是异常体系的根，它继承自`Object`。`Throwable`有两个体系：`Error`和`Exception`，`Error`表示严重的错误，程序对此一般无能为力，例如：

- `OutOfMemoryError`：内存耗尽
- `NoClassDefFoundError`：无法加载某个Class
- `StackOverflowError`：栈溢出

而`Exception`则是运行时的错误，它可以被捕获并处理。

某些异常是应用程序逻辑处理的一部分，应该捕获并处理。例如：

- `NumberFormatException`：数值类型的格式错误
- `FileNotFoundException`：未找到文件
- `SocketException`：读取网络失败

还有一些异常是程序逻辑编写不对造成的，应该修复程序本身。例如：

- `NullPointerException`：对某个`null`的对象调用方法或字段
- `IndexOutOfBoundsException`：数组索引越界

`Exception`又分为两大类：

1. `RuntimeException`以及它的子类；
2. 非`RuntimeException`（包括`IOException`、`ReflectiveOperationException`等等）

Java规定：

- 必须捕获的异常，包括`Exception`及其子类，但不包括`RuntimeException`及其子类，这种类型的异常称为 Checked Exception。
- 不需要捕获的异常，包括`Error`及其子类，`RuntimeException`及其子类。

##### 相关文章

[1] [Java的异常 - 廖雪峰的官方网站 (liaoxuefeng.com)](https://www.liaoxuefeng.com/wiki/1252599548343744/1264734349295520)



### 反射



### 集合















