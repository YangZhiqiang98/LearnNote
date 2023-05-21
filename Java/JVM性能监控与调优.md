## 性能监控与调优

**使用数据说明问题，使用知识分析问题，使用工具处理问题**。

**无监控，不调优**



### 简单命令行工具



#### jps: 查看正在运行的 Java 进程

[Java命令：jps — 查看进程信息_jps -l_蹲街式等待的博客-CSDN博客](https://blog.csdn.net/wangzhongshun/article/details/112546027)

| 参数 | 说明                            |
| ---- | ------------------------------- |
| `-l` | 输出主类全名或jar路径           |
| `-q` | 只输出LVMID                     |
| `-m` | 输出JVM启动时传递给main()的参数 |
| `-v` | 输出JVM启动时显示指定的JVM参数  |

**jps 失效**

1、如果 Java 进程关闭了默认开启的 UsePerfData 参数（即使用参数 -XX:-UsePerfData），那么 jps 无法探知该进程。

2、jps 实现机制是在目录（/tmp/hsperfdata_{userName}/）下生成几个文件，文件名就是 java 进程的 pid。 jps 列出进程 id 就是列出该目录下的文件名，系统参数，则是读取文件中的内容。

- 磁盘满，无法创建文件。
- 用户无读取权限
- 文件或者目录被清除

#### jstat:查看 JVM 的统计信息

[Java命令：jstat — 查看JVM的GC信息_jstat gc_蹲街式等待的博客-CSDN博客](https://davis.blog.csdn.net/article/details/112545871)



```
 jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```



-t : 输出信息前加一个 timestamp 列， 显示程序运行的时间，单位秒

```
C:\Users\23919>jstat -compiler -t 12412 1000 5
Timestamp       Compiled Failed Invalid   Time   FailedType FailedMethod
          267.9      256      0       0     0.07          0
          269.0      256      0       0     0.07          0
          270.0      256      0       0     0.07          0
          271.0      256      0       0     0.07          0
          272.0      256      0       0     0.07          0
```

-h: 周期性数据输出后，多少行数据打印一个表头信息

```
C:\Users\23919>jstat -compiler -h3 -t 12412 1000 5
Timestamp       Compiled Failed Invalid   Time   FailedType FailedMethod
          433.4      256      0       0     0.07          0
          434.4      256      0       0     0.07          0
          435.4      256      0       0     0.07          0
Timestamp       Compiled Failed Invalid   Time   FailedType FailedMethod
          436.4      256      0       0     0.07          0
          437.4      256      0       0     0.07          0
```



#### jinfo: 查看进程参数

[Java命令：jinfo — 查看进程参数_查看java启动参数_蹲街式等待的博客-CSDN博客](https://davis.blog.csdn.net/article/details/122298396)



并非所有的参数都支持动态修改，参数只有被标记为 manageable 的 flag 可以被实时修改。

查看被标记为 manageable 的参数。

```
java -XX:+PrintFlagsFinal -version | grep manageable
```



java -XX:+PrintFlagsFinal : 查看 JVM 参数最终值

java -XX:+PrintFlagsInitial: 查看所有 JVM 参数启动的初始值

java -XX:+PrintCommandLineFlags: 查看那些已经被用户或者 JVM  设置过的详细的 xx 参数的名称和值



#### jmap:导出内存映像&内存使用情况

[Java命令：jmap — 打印指定进程的共享对象内存映射或堆内存细节_jmap 打印对象_蹲街式等待的博客-CSDN博客](https://davis.blog.csdn.net/article/details/103017723)



#### jhat: 堆分析工具

在 jdk8 之上已经被删除，建议使用 VisualVm 代替



#### jstack：线程快照

[Java命令：jstack — 获取线程dump信息_jstack dump_蹲街式等待的博客-CSDN博客](https://davis.blog.csdn.net/article/details/104632680)



#### jcmd: 多功能命令行

推荐







