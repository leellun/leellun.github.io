---
title: JVM优化命令及代码分析
date: 2019-01-08 20:14:02
categories:
  - 后端
  - java
tags:
  - jvm 
---

# 1 jvm 优化

## 1.1 JDK优化命令

### 1.1.2 jps

| 选项 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| -q   | 只显示pid，不显示class名称,jar文件名和传递给main方法的参数   |
| -m   | 输出传递给main方法的参数，在嵌入式jvm上可能是null            |
| -l   | 输出应用程序main class的完整package名或者应用程序的jar文件完整路径名 |
| -v   | 输出传递给JVM的参数(线程id、执行线程的主类名和JVM配置信息)   |
| -V   | 隐藏输出传递给JVM的参数                                      |

### 1.1.2 jstat

用于提供与JVM性能相关的统计信息，例如垃圾收集，编译活动。 jstat的主要优势在于，它可以在运行JVM且无需任何先决条件的情况下动态捕获这些指标。 

jstat -选项 线程id 执行时间（单位毫秒） 执行次数 

| 选项            | 说明                           |
| --------------- | ------------------------------ |
| -class          | 显示ClassLoad的相关信息；      |
| -compiler       | 显示JIT编译的相关信息；        |
| -gc             | 显示和gc相关的堆信息；         |
| -gccapacity     | 显示各个代的容量以及使用情况； |
| -gcmetacapacity | 显示metaspace的大小            |
| -gcnew          | 显示新生代信息；               |
| -gcnewcapacity  | 显示新生代大小和使用情况；     |
| -gcold          | 显示老年代和永久代的信息；     |
| -gcoldcapacity  | 显示老年代的大小；             |

### 1.1.3 jinfo

java配置信息工具，实时查看和调整虚拟机各项参数。使用jps命令的-v参数可以查看虚拟机启动时显示指定的参数列表，jinfo -flag可以进行查询，java -XX:+PrintFlagsFinal查看参数默认值

用法：jinfo [option] <pid>  

例如：jinfo -flags CMSInitiatingOccupancyFraction 88666  结果：-XX:CMSInitiatingOccupancyFraction=-1

| 选项                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| -flag <name>         | 用于打印虚拟机标记参数的值，name表示虚拟机标记参数的名称。   |
| -flag [+\|-]<name>   | 用于开启或关闭虚拟机标记参数。+表示开启，-表示关闭。         |
| -flag <name>=<value> | 用于设置虚拟机标记参数，但并不是每个参数都可以被动态修改的。 |
| -flags               | 打印虚拟机参数。                                             |
| -sysprops            | 打印系统参数。                                               |

### 1.1.4 jmap

java内存映射工具。

命令：jmap -heap pid  描述：显示Java堆详细信息 

jmap [选项] pid

| 选项                 | 说明                                                   |
| -------------------- | ------------------------------------------------------ |
| no option            | 查看进程的内存映像信息,类似 Solaris pmap 命令。        |
| -heap                | 显示Java堆详细信息                                     |
| -histo[:live]        | 显示堆中对象的统计信息                                 |
| -clstats             | 打印类加载器信息                                       |
| -dump:<dump-options> | 生成堆转储快照 jmap -dump:format=b,file=ecli.bin 88666 |
| -J<flag>             | 指定传递给运行jmap的JVM的参数                          |

### 1.1.5 jhat

虚拟机转存快照分析工具，一般不用此命令。

jhat 快照文件.dump

### 1.1.6 jvisualvm

一个JDK内置的图形化VM监视管理工具 

### 1.1.7 visualgc插件

### 1.1.8 jprofile

### 1.1.9  查询堆信息

```
jhsdb jmap --heap --pid  62784
```

```
Parallel GC with 13 thread(s)   #13个gc线程  
  
Heap Configuration:#堆内存初始化配置  
   MinHeapFreeRatio = 40  #-XX:MinHeapFreeRatio设置JVM堆最小空闲比率  
   MaxHeapFreeRatio = 70  #-XX:MaxHeapFreeRatio设置JVM堆最大空闲比率  
   MaxHeapSize      = 8436842496 (8046.0MB)#-XX:MaxHeapSize=设置JVM堆的最大大小  
   NewSize          = 5439488 (5.1875MB) #-XX:NewSize=设置JVM堆的‘新生代’的默认大小  
   MaxNewSize       = 17592186044415 MB  #-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小  
   OldSize          = 5439488 (5.1875MB) #-XX:OldSize=设置JVM堆的‘老生代’的大小  
   NewRatio         = 2 #-XX:NewRatio=:‘新生代’和‘老生代’的大小比率  
   SurvivorRatio    = 8 #-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值  
   MetaspaceSize            = 21807104 (20.796875MB) 元数据区初始大小
   CompressedClassSpaceSize = 1073741824 (1024.0MB) 存放 Klass 对象，需要一个连续的不超过 4G 的内存
   MaxMetaspaceSize         = 17592186044415 MB  元数据区最大大小
   G1HeapRegionSize         = 1048576 (1.0MB) G1垃圾回收期，设置每个Region的大小。值是2的幂，范围是1MB到32MB之间，目标是根据最小的Java堆大小划分出约2048个区域。默认是堆内存的1/2000。
  
Heap Usage:
G1 Heap:
   regions  = 512  一整块内存空间,被分为多个heap区
   capacity = 536870912 (512.0MB) 容积
   used     = 95456968 (91.03485870361328MB)  使用
   free     = 441413944 (420.9651412963867MB) 空闲
   17.78024584054947% used
PS Young Generation  
Eden Space:#Eden区内存分布  
   regions  = 8  内存块
   capacity = 282066944 (269.0MB)  容量
   used     = 8388608 (8.0MB) 
   free     = 273678336 (261.0MB)
   2.973977695167286% used
Survivor Space:#其中一个Survivor区的内存分布  
   regions  = 0
   capacity = 0 (0.0MB)
   used     = 0 (0.0MB)
   free     = 0 (0.0MB)
   0.0% used
G1 Old Generation: 老年代
   regions  = 87
   capacity = 254803968 (243.0MB)
   used     = 87068360 (83.03485870361328MB)
   free     = 167735608 (159.96514129638672MB)
   34.170723746342915% used
```

查看java堆上的对象分布情况 :

```
jhsdb jmap --histo  --pid   62784
```

# 2 代码分析

Github：https://github.com/leelun/jvmdemo

## 2.1 查看GC收集器

```java
List<GarbageCollectorMXBean> lists=ManagementFactory.getGarbageCollectorMXBeans();
for(GarbageCollectorMXBean bean:lists){
	System.out.println(bean.getName());
}
```

## 2.2 JVM内存配置及分析

JVM参数设置：

```
-Xms5m -Xmx20m -XX:+PrintGCDetails -XX:+UseSerialGC -XX:+PrintCommandLineFlags
```

代码：main()方法

```java
//申请1M内存
byte[] b1=new byte[110241024];
//申请5M内存
byte[] b2=new byte[510241024];
//申请5M内存
byte[] b3=new byte[510241024];
```

GC打印信息：

```
1 [GC (Allocation Failure) DefNew: 759K->127K(1152K), 0.0013231 secs 759K->533K(1920K), [Metaspace: 2657K->2657K(1056768K)], 0.0028295 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2 [GC (Allocation Failure) DefNew: 19K->0K(1152K), 0.0001956 secs 1577K->1557K(2948K), [Metaspace: 2657K->2657K(1056768K)], 0.0015740 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
3 [GC (Allocation Failure) DefNew: 21K->0K(1216K), 0.0002020 secs 6698K->6677K(8936K), [Metaspace: 2657K->2657K(1056768K)], 0.0016114 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

注意：这里前两个内存申请做分析

内存阶段分析：

**阶段1：申请1M内存空间分析（申请1M之前、申请失败后GC回收空间、申请1M空间之后）**

(1)申请1M空间之前  新生代759k(1152k) 老年代405k(768k) 堆内存759k(1920k) Metaspace2657K(1056768K)

(2)申请1M内存失败(Allocation Failure)，GC回收空间新生代759K->127K(1152K)，老年代405K->533K(768K)，堆内存759K->533K(1920K)及Metaspace: 2657K->2657K(1056768K)

(3)申请1M空间后内存变化 新生代19K(1152K) 老年代1557K(1796K) 堆内存1577K(2948K) Metaspace2657K(1056768K)

**阶段2：申请5M内存空间分析**

(1)申请5M空间之前 新生代19K(1152k) 老年代1557K(1796K) 堆内存1577K(2948K) Metaspace2657K(1056768K)

(2)申请5M内存失败(Allocation Failure)，GC回收空间新生代19K->0K(1152K)，老年代1557K->1557K(1796K)，堆内存1577K->1557K(2948K)及Metaspace: 2657K->2657K(1056768K)

(3)申请5M空间后内存变化 新生代21K(1216K) 老年代6677K(7720K) 堆内存6698K(8936K) Metaspace2657K(1056768K)

**阶段3：申请5M内存空间分析**

这里就不做分析了

**下面是添加-XX:+PrintGCDetails后的打印消息**

```
Heap
 def new generation  total 5120K, used 182K [0x00000000fec00000, 0x00000000ff180000, 0x00000000ff2a0000)
 eden space 4608K,  3% used [0x00000000fec00000, 0x00000000fec2d8a0, 0x00000000ff080000)
 from space 512K,  0% used [0x00000000ff080000, 0x00000000ff080000, 0x00000000ff100000)
 to  space 512K,  0% used [0x00000000ff100000, 0x00000000ff100000, 0x00000000ff180000)
 tenured generation  total 13696K, used 11797K [0x00000000ff2a0000, 0x0000000100000000, 0x0000000100000000)
  the space 13696K,  86% used [0x00000000ff2a0000, 0x00000000ffe25608, 0x00000000ffe25800, 0x0000000100000000)
 Metaspace    used 2664K, capacity 4486K, committed 4864K, reserved 1056768K
 class space   used 288K, capacity 386K, committed 512K, reserved 1048576K
```

分析：

新生代 空间5120k 使用182k

Eden 空间4608K 3%使用率

From(s0) 空间512K 0%使用率

To(s1) 空间512k 0%使用率

Tenured 空间13696K 使用11797k 使用率86%

计算分析：

新生代

0x00000000ff180000-0x00000000fec00000=5632K

5632k=eden+from+to  因为from和to使用复制算法的结构，当一方有内存使用时，另一方空闲。所以 new generation 总大小=5632k- (from或者to)

Eden from to 总大小=第三个参数-第一个参数  使用大小=第二个参数-第一个参数





# 附表：

## JVM常见参数

https://blog.csdn.net/huaxin_sky/article/details/79435599

| 选项和默认值                                 | 描述                                                         |
| -------------------------------------------- | ------------------------------------------------------------ |
| -XX:+PrintGC                                 | 虚拟机启动后，遇到GC就会打印日志                             |
| -XX:+PrintGCDetails                          | 查看详细信息，包括各个区的信息                               |
| -Xms:初始堆大小                              | JVM启动的时候，给定堆空间大小。                              |
| -Xmx:最大堆大小                              | JVM运行过程中，如果初始堆空间不足的时候，最大可以扩展到多少。 |
| -XX:+PrintCommandLineFlags                   | 可以将隐式或者显式传给虚拟机的参数输出                       |
| -Xmn：设置年轻代大小。                       | 整个堆大小**=**年轻代大小**+**年老代大小**+**持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。（java8已无持久代） 相当于下面同时设置：-XX:MaxNewSize -XX:NewSize |
| -Xss： 设置每个线程的Java栈大小。            | JDK5.0以后每个线程Java栈大小为1M，以前每个线程堆栈大小为256K。根据应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。 |
| -XX:NewRatio=n                               | 设置年轻代和年老代的比值。如:为3,表示年轻代与年老代比值为1：3，年轻代占整个年轻代+年老代和的1/4 |
| -XX:SurvivorRatio=n                          | 年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5 |
| -XX:MaxPermSize=n                            | 设置持久代大小(java8已弃用)                                  |
| -XX:MaxTenuringThreshold：设置垃圾最大年龄。 | 如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。 |
| -XX:MaxDirectMemorySize 默认为-Xms大小       | 直接内存配置参数，直接内存达到上限会触发GC，如果有效的释放空间，也会出现OOM |
| -XX:MaxTenuringThreshold                     | 控制新生代对象的最大年龄，当超过这个年龄范围就会晋升老年代，默认情况下15. |
| -XX:PretenureSizeThreshold                   | 设置对象的大小超过在指定的大小之后，直接晋升老年代。但是TLAB区域优先分配内存。 |
| -XX:+UseTLAB                                 | TLAB（Thread Local Allocation Buffer）:线程本地分配缓存，一个线程专用的内存分配区域，为了加速对象分配而生。每一个线程都会产生一个TLAB，该线程独享的工作区域，java虚拟机使用TLAB区来避免使用多线程冲突问题，提高了对象分配的效率。TLAB空间一般不会太大，当大对象无法在TLAB分配时，则会直接分配到堆上。 |
| -XX:TLABSize                                 | TLAB大小                                                     |
| -XX:TLABRefillWasteFraction                  | 设置维护进入TLAB空间的单个对象的大小，默认64，如果对象大于整个空间的1/64，则在堆创建对象。 |
| -XX:+PrintTLAB                               | 查看TLAB信息                                                 |
| -XX:ResizeTLAB                               | 自调整TLABRefillWasteFraction阈值                            |
| -XX:TLABWasteTargetPercent                   | TLAB占eden区的百分比                                         |
| XX:HeapDumpPath=./java_pid<pid>.hprof        | 指定HeapDump的文件路径或目录                                 |
| -XX:-HeapDumpOnOutOfMemoryError              | 当抛出OOM时进行HeapDump                                      |
| -XX:MaxDirectMemorySize                      | 直接内存大小，默认与Java堆最大值(-Xmx)一样                   |
| -XX:+UseSerialGC                             | 配置串行回收器 Serial                                        |
| -XX:+UseParallelGC                           | Parallel Scavenge                                            |
| -XX:-UseParallelOldGC                        | Serial Old                                                   |
| -XX:+UseParNewGC                             | Parallel New                                                 |
| -XX:+UseConcMarkSweepGC                      | CMS                                                          |
| -XX:+UseG1GC                                 | G1                                                           |
| XX:+DisableExplicitGC                        | 忽略System.gc()                                              |
| -XX:MaxNewSize=size                          | 新生代占整个堆内存的最大值                                   |
| -XX:NewSize=size                             | 新生代预估上限的默认值                                       |
| -XX:MaxHeapSize                              | 等价于-Xmx                                                   |
| -XX:MetaspaceSize                            | 这个参数是初始化的Metaspace大小                              |
| -XX:MaxMetaspaceSize=size                    | Metaspace内存最大值                                          |
| -XX:MinMetaspaceFreeRatio=N                  | 如果空闲比小于这个参数，那么虚拟机将增长Metaspace的大小。设置该参数可以控制Metaspace的增长的速度，太小的值会导致Metaspace增长的缓慢，Metaspace的使用逐渐趋于饱和，可能会影响之后类的加载。而太大的值会导致Metaspace增长的过快，浪费内存。 |
| -XX:MaxMetasaceFreeRatio=N                   | 如果空闲比大于这个参数，那么虚拟机会释放Metaspace的部分空间。 |
| -XX:MaxMetaspaceExpansion=N                  | Metaspace增长时的最大幅度。                                  |
| -XX:MinMetaspaceExpansion=N                  | Metaspace增长时的最小幅度。                                  |
| -XX:+ExplicitGCInvokesConcurrent             | Full gc有更少的停机时间  并行full gc                         |