## Scope的分类

- **compile**:如果没有指定scope,此值是默认值，编译依赖将会在所有的类路径中可用，同时它们会被打包。此外，这些dependency会传递到依赖的项目中。
- **test**: 此依赖在一般的编译和运行时都不需要，只有在测试编译和测试运行时可用。也不是传递性的。
- **provided**: 功能和Compile很像，但是也说明此依赖仅在编译和测试类路径上可用，运行时不打包，此依赖由JDK或者容器提供。也不是传递性的
- **runtime**: 此依赖在运行和测试系统的时候需要，但在编译时不需要。比如可能在编译的时候只需要JDBC API JAR，而只有在运行的时候才需要JDBC驱动实现。
- **system**：与provided类似，不过被依赖项不会从maven仓库抓，而是从本地文件系统拿，一定需要配合systemPath属性使用。

## scope的依赖传递

A–>B–>C。当前项目为A，A依赖于B，B依赖于C。知道B在A项目中的scope，那么怎么知道C在A中的scope呢？答案是： 
当C是test或者provided时，C直接被丢弃，A不依赖C； 否则A依赖C，C的scope继承于B的scope。



## 参考文档

[1] [Maven – POM Reference (apache.org)](https://maven.apache.org/pom.html#Dependency_Version_Requirement_Specification)

[2] [maven中pom文件中scope的作用 - yvioo - 博客园 (cnblogs.com)](https://www.cnblogs.com/pxblog/p/12585445.html)

[3] [pom文件中scope标签基本解释_weixin_30693183的博客-CSDN博客](https://blog.csdn.net/weixin_30693183/article/details/95225530)

