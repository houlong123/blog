---
title: JVM额外知识学习
date: 2018-04-01 21:22:32
tags: java
---

#### GC日志分析

```
0.303: [Full GC (System.gc()) [PSYoungGen: 464K->0K(76288K)] [ParOldGen: 8K->428K(175104K)] 472K->428K(251392K), 
[Metaspace: 3086K->3086K(1056768K)], 0.0058612 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
```

##### 分析

1. <font color=red>0.303</font>：表示 <font color=red>**GC发生的时间**</font>，从java虚拟机启动以来经过的秒数。

2. <font color=red>[Full GC (System.gc()) </font>：表示 <font color=red>**垃圾收集的停顿类型**</font>。有 **`[GC `, `[Full` , `[Full GC(System.gc())`类型**。 其中，有 [Full 说明GC过程中发生了STW。如果调用了System.gc()方法触发了收集，则显示：[Full GC(System.gc())

3. <font color=red>[PSYoungGen</font>：表示 <font color=red>**GC发生区域**</font>。有 `[DefNew`、`[Tenured`、`[Perm`等类型。 此处显示的区域名称与使用GC的垃圾收集器有关。<font color=gray>Serial收集器中的新生代名为“Default New Generation”，所以显示的是“[DefNew”，如果是ParNew收集器，新生代名称就会变为“[ParNew”，意为“Parallel New Generation”。如果采用Parallel Scavenge收集器，那它配套的新生代称为“PSYoungGen”，老年代和永久代同理，名称也是由收集器决定的。</font>

4. <font color=red>464K->0K(76288K)</font>：表示 <font color=red>**GC前该内存区域已使用容量-＞GC后该内存区域已使用容量（该内存区域总容量）**</font>

<!-- more -->

5. <font color=red>472K->428K(251392K)</font>：表示<font color=red>**GC前Java堆已使用容量-＞GC后Java堆已使用容量（Java堆总容量）**</font>

6. <font color=red>0.0058612 secs</font>：表示 <font color=red>**该内存区域GC所占用的时间，单位是秒。**</font>

7. <font color=red>[Times: user=0.01 sys=0.00, real=0.00 secs] </font>：此处 user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU时间和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。<font color=gray>CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。</font>


---

#### GC故障排查工具
当应用部署到生成环境后，无论是直接接触物理服务器还是Telnet到服务器上有可能受到限制。通过java提供的用于<font color=red>**监控虚拟机和故障处理的工具**</font>，我们可以直接在应用程序中实现功能强大的监控分析功能。JDK主要命令行监控工具有：

名称 | 主要作用
---|---
jps | JVM Process Status Tool。<font color=red>**显示指定系统内所有的HotSpot虚拟机进程**</font>
jstat | JVM Statistics Monitoring Tool。<font color=red>**用于收集HotSpot虚拟机各方面的运行数据**</font>
jinfo | Configuration Info for Java。<font color=red>**显示虚拟机配置信息**</font>
jmap | Memory Map for Java。<font color=red>**生成虚拟机的内存转储快照（headdump文件）**</font>
jhat | JVM Heap Dump Browser。<font color=red>**用于分析heapdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果**</font>
jstack | Stack Trace for Java。<font color=red>**显示虚拟机的线程快照**</font>

##### 工具详解

###### jps：虚拟机进程状况工具
+ 意义
    
    可以<font color=red>**列出正在运行的虚拟机进程**</font>，并显示虚拟机执行主类名称（Main Class，main()函数所在类）以及进程的本地虚拟机唯一ID（Local Virtual Machine Identifier，LVIMD）。

+ 命令格式

    <font color=red>**jps [options] [hostid]**</font>
    
    其中，`hostid` 为RMI注册表中注册的主机名。`options` 可选参数有：
    

选项 | 作用
---|---
-q | 只输出LVIMD，省略主类的名称
-m | 输出虚拟机进程启动时传递给main()函数的参数
-l | 输出主类的全名，如果进程执行的是jar包，输出jar路径
-V | 输出主类的简名
-v | 输出虚拟机进程启动时的JVM参数

+ 实例

1. jps -q  <font color=gray>**（只输出所有虚拟机进程ID）**</font>
```
houlongdeMacBook-Pro:JavaPattern houlong$ jps -q
22802
22803
23125
11685
```

2. jps -m  <font color=gray>**（输出虚拟机进程启动时传递给主类main()函数的参数）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jps -m
22802 Launcher /Applications/IntelliJ IDEA.app/Contents/lib/javac2.jar:/Applications/IntelliJ IDEA.app/Contents/lib/snappy-in-java-0.5.1.jar:/Applications/IntelliJ IDEA.app/Contents/lib/jna.jar:/Applications/IntelliJ IDEA.app/Contents/lib/openapi.jar:/Applications/IntelliJ IDEA.app/Contents/lib/oromatcher.jar:/Applications/IntelliJ IDEA.app/Contents/lib/trove4j.jar:/Applications/IntelliJ IDEA.app/Contents/lib/netty-all-4.1.5.Final.jar:/Applications/IntelliJ IDEA.app/Contents/lib/jps-builders.jar:/Applications/IntelliJ IDEA.app/Contents/lib/nanoxml-2.2.3.jar:/Applications/IntelliJ IDEA.app/Contents/lib/jgoodies-forms.jar:/Applications/IntelliJ IDEA.app/Contents/lib/jdom.jar:/Applications/IntelliJ IDEA.app/Contents/lib/asm-all.jar:/Applications/IntelliJ IDEA.app/Contents/lib/protobuf-2.5.0.jar:/Applications/IntelliJ IDEA.app/Contents/lib/rt/jps-plugin-system.jar:/Applications/IntelliJ IDEA.app/Contents/lib/jna-platform.jar:/Applications/IntelliJ IDEA.app/Contents/lib/annotations.jar:/Applica
22803 AppMain com.houlong.java.gc.StackOverFlowTest
11685 
23133 Jps -m
```

3. jps -l  <font color=gray>**（输出主类的全名）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jps -l
22802 org.jetbrains.jps.cmdline.Launcher
22803 com.intellij.rt.execution.application.AppMain
11685 
23142 sun.tools.jps.Jps
```

4. jps -V  <font color=gray>**（输出主类的简名）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jps -V
22802 Launcher
22803 AppMain
11685 
23151 Jps
```

5. jps -v  <font color=gray>**（输出虚拟机进程启动时JVM参数）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jps -v
22802 Launcher -Xmx700m -Djava.awt.headless=true -Djava.endorsed.dirs="" -Djdt.compiler.useSingleThread=true -Dpreload.project.path=/Users/houlong/workplace/JavaPattern -Dpreload.config.path=/Users/houlong/Library/Preferences/IntelliJIdea2016.3/options -Dcompile.parallel=false -Drebuild.on.dependency.change=true -Djava.net.preferIPv4Stack=true -Dio.netty.initialSeedUniquifier=2464894803496615857 -Dfile.encoding=UTF-8 -Djps.file.types.component.name=FileTypeManager -Duser.language=zh -Duser.country=CN -Didea.paths.selector=IntelliJIdea2016.3 -Didea.home.path=/Applications/IntelliJ IDEA.app/Contents -Didea.config.path=/Users/houlong/Library/Preferences/IntelliJIdea2016.3 -Didea.plugins.path=/Users/houlong/Library/Application Support/IntelliJIdea2016.3 -Djps.log.dir=/Users/houlong/Library/Logs/IntelliJIdea2016.3/build-log -Djps.fallback.jdk.home=/Applications/IntelliJ IDEA.app/Contents/jdk/Contents/Home/jre -Djps.fallback.jdk.version=1.8.0_112-release -Djava.io.tmpdir=/Users/houlong/Library/Caches/IntelliJIdea2016.3/compile-se
23155 Jps -Dapplication.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_111.jdk/Contents/Home -Xms8m
22803 AppMain -XX:MetaspaceSize=5m -XX:MaxMetaspaceSize=5m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/houlong/workplace/JavaPattern/headDump.log -Xloggc:/Users/houlong/workplace/JavaPattern/hha.log -Didea.launcher.port=7535 -Didea.launcher.bin.path=/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
11685  -Dfile.encoding=UTF-8 -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Xverify:none -XX:ErrorFile=/Users/houlong/java_error_in_idea_%p.log -XX:HeapDumpPath=/Users/houlong/java_error_in_idea.hprof -Xbootclasspath/a:../lib/boot.jar -Xms128m -Xmx1500m -XX:ReservedCodeCacheSize=500m -XX:+UseCompressedOops -XX:MaxPermSize=500m -Djb.vmOptionsFile=/Users/houlong/Library/Preferences/IntelliJIdea2016.3/idea.vmoptions -Didea.java.redist=jdk-bundled -Didea.home.path=/Applications/IntelliJ IDEA.app/Contents -Didea.executable=idea -Didea.paths.selector=IntelliJIdea2016.3
```


###### jstat：虚拟机统计信息监控工具
+ 意义

    用于 <font color=red>**监视虚拟机各种运行状态信息的命令行工具**</font> 。它可以显示本地或者远程虚拟机进程中的类装载，内存，垃圾收集，JIT编译等运行数据。
    
+ 命令格式

    <font color=red>**jstat [-option] [vmid] [interval(间隔时间)/毫秒] [count(查询次数)]**</font>
    
    表示每隔interval时间后，查询count次JVM的相应信息。
    对于命令格式中的 VMID 和 LVMID ，如果是本地虚拟机进程，则二者是一致的；如果是远程虚拟机进程，则 VMID 的格式为：
    `[protocol:][//]lvimd[@hostname[:port]/servername]`
    
    
```
jstat -gc 2764 1000 20
表示每1s中查询进程2764的垃圾收集状况，一共查询20次
```
其中，`option`可选参数有：

选项 |  作用
---|---
-class | 监视类装载，卸载数量，总空间以及类装载所耗费的时间
-gc | 监视java堆状况，包括Eden区，两个Survivor区，老年代，方法区等容量，已用空间，GC时间合计等信息
-gccapacity| 监视内容与-gc基本相同，但输出<font color=red>**主要关注java堆各个区域使用到的最大，最小空间**</font>
-gcutil | 监视内容与-gc基本相同，但输出<font color=red>**主要关注已使用空间占总空间的百分比**</font>
-gccause | 与-gcutil功能一样，但是<font color=red>**会额外输出导致上一次GC产生的原因**</font>
-gcnew | 监视新生代GC状况
-gcnewcapacity | 监视内容与-gcnew 基本相同，输出<font color=red>**主要关注使用到的最大，最小空间**</font>
-gcold | 监视老年代GC状况
-gcoldcapacity |  监视内容与-gcold 基本相同，输出<font color=red>**主要关注使用到的最大，最小空间**</font>
-gcmetacapacity | 监视元数据空间GC状况，输出<font color=red>**主要关注使用到的最大，最小空间**</font>
-compiler | 输出JIT编译器编译过的方法，耗时等信息
-printcompilation| 输出已被JIT编译的方法

+ 实例

1. jstat -class <font color=gray>**（类加载统计。监视类装载，卸载数量，总空间以及类装载所耗费的时间）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -class 22803 1s 2
Loaded  Bytes  Unloaded  Bytes     Time   
   527  1059.7        0     0.0       0.25
   527  1059.7        0     0.0       0.25

解释：   
Loaded：加载class的数量
Bytes：所占用空间大小
Unloaded：未加载数量
Bytes：未加载占用空间
Time：类装载所耗费的时间
```

2. jstat -compiler<font color=gray>**（编译统计。输出JIT编译器编译过的方法，耗时等信息）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -compiler 24528
Compiled Failed Invalid   Time   FailedType FailedMethod
    5454      1       0    30.49          1 org/springframework/core/annotation/AnnotationUtils findAnnotation
    
解释：
Compiled：编译数量。
Failed：失败数量
Invalid：不可用数量
Time：时间
FailedType：失败类型
FailedMethod：失败的方法
```

3. jstat -gc<font color=gray>**（垃圾回收统计。监视java堆状况）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gc 24528 1s 2
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
16896.0 16384.0  0.0   13231.7 248832.0 70339.6   134656.0   27791.6   52696.0 50328.3 7128.0 6696.4     11    0.178   2      0.114    0.291
16896.0 16384.0  0.0   13231.7 248832.0 70339.6   134656.0   27791.6   52696.0 50328.3 7128.0 6696.4     11    0.178   2      0.114    0.291

解释：
S0C：第一个Survivor区的大小
S1C：第二个Survivor区的大小
S0U：第一个Survivor区的使用大小
S1U：第二个Survivor区的使用大小
EC：Eden区的大小
EU：Eden区的使用大小
OC：老年代大小
OU：老年代使用大小
MC：元数据区大小
MU：元数据区使用大小
CCSC：压缩类空间大小
CCSU：压缩类空间使用大小
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

4. jstat -gccapacity<font color=gray>**（堆内存统计。主要关注java堆各个区域使用到的最大，最小空间）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gccapacity 24528 1s 2
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC 
 87040.0 1397760.0 396800.0 16896.0 16384.0 248832.0   175104.0  2796544.0   134656.0   134656.0      0.0 1095680.0  52696.0      0.0 1048576.0   7128.0     11     2
 87040.0 1397760.0 396800.0 16896.0 16384.0 248832.0   175104.0  2796544.0   134656.0   134656.0      0.0 1095680.0  52696.0      0.0 1048576.0   7128.0     11     2

解释：
NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0C：第一个Survivor区大小
S1C：第二个Survivor区大小
EC：Eden区大小
OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC：当前老年代大小
MCMN：最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代gc次数
FGC：老年代GC次数
```

5. jstat -gcutil<font color=gray>**（总结垃圾回收统计。主要关注java堆各个区域已使用空间占总空间的百分比）**</font>


```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcutil 24528 1s 2
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00  80.76  31.80  20.64  95.51  93.94     11    0.178     2    0.114    0.291
  0.00  80.76  31.80  20.64  95.51  93.94     11    0.178     2    0.114    0.291
  
解释：
S0：第一个Survivor区当前使用比例
S1：第二个Survivor区当前使用比例
E：Eden区使用比例
O：老年代使用比例
M：元数据区使用比例
CCS：压缩使用比例
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

6. jstat -gccause<font color=gray>**（总结垃圾回收统计。与-gcutil基本一致，只是额外输出了上次导致GC的原因）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gccause 24528 1s 2
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
  0.00  80.76  32.60  20.64  95.51  93.94     11    0.178     2    0.114    0.291 Allocation Failure   No GC               
  0.00  80.76  32.60  20.64  95.51  93.94     11    0.178     2    0.114    0.291 Allocation Failure   No GC               

```

7. jstat -gcnew<font color=gray>**（新生代垃圾回收统计。监视新生代GC状况）**</font>


```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcnew 24528 1s 2
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT  
16896.0 16384.0    0.0 13231.7  1  15 16896.0 248832.0  83113.4     11    0.178
16896.0 16384.0    0.0 13231.7  1  15 16896.0 248832.0  83113.4     11    0.178

解释：
S0C：第一个Survivor区大小
S1C：第二个Survivor区的大小
S0U：第一个Survivor区的使用大小
S1U：第二个Survivor区的使用大小
TT：对象在新生代存活的次数
MTT：对象在新生代存活的最大次数
DSS：期望的幸存区大小
EC：Eden区的大小
EU：Eden区的使用大小
YGC：年轻代垃圾回收次数
YGCT：年轻代垃圾回收消耗时间
```

8. jstat -gcnewcapacity<font color=gray>**（新生代内存统计。监视新生代GC状况）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcnewcapacity 24528 1s 2
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC 
   87040.0  1397760.0   396800.0 465920.0  16896.0 465920.0  16384.0  1396736.0   248832.0    11     2
   87040.0  1397760.0   396800.0 465920.0  16896.0 465920.0  16384.0  1396736.0   248832.0    11     2

解释：
NGCMN：新生代最小容量
NGCMX：新生代最大容量
NGC：当前新生代容量
S0CMX：最大幸存1区大小
S0C：当前幸存1区大小
S1CMX：最大幸存2区大小
S1C：当前幸存2区大小
ECMX：最大伊甸园区大小
EC：当前伊甸园区大小
YGC：年轻代垃圾回收次数
FGC：老年代回收次数
```

9. jstat -gcold<font color=gray>**（老年代垃圾回收统计。监视老年代GC状况）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcold  24528 1s 2
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT   
 52696.0  50328.3   7128.0   6696.4    134656.0     27791.6     11     2    0.114    0.291
 52696.0  50328.3   7128.0   6696.4    134656.0     27791.6     11     2    0.114    0.291

解释：
MC：方法区大小
MU：方法区使用大小
CCSC：压缩类空间大小
CCSU：压缩类空间使用大小
OC：老年代大小
OU：老年代使用大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

10. jstat -gcoldcapacity<font color=gray>**（老年代内存统计。主要关注使用到的最大，最小空间）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcoldcapacity  24528 1s 2
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT   
   175104.0   2796544.0    134656.0    134656.0    11     2    0.114    0.291
   175104.0   2796544.0    134656.0    134656.0    11     2    0.114    0.291

解释：
OGCMN：老年代最小容量
OGCMX：老年代最大容量
OGC：当前老年代大小
OC：老年代大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

11. jstat -gcmetacapacity<font color=gray>**（元数据空间统计。主要关注使用到的最大，最小空间）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -gcmetacapacity 24528 1s 2
   MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT   
       0.0  1095680.0    52696.0        0.0  1048576.0     7128.0    11     2    0.114    0.291
       0.0  1095680.0    52696.0        0.0  1048576.0     7128.0    11     2    0.114    0.291

解释：
MCMN:最小元数据容量
MCMX：最大元数据容量
MC：当前元数据空间大小
CCSMN：最小压缩类空间大小
CCSMX：最大压缩类空间大小
CCSC：当前压缩类空间大小
YGC：年轻代垃圾回收次数
FGC：老年代垃圾回收次数
FGCT：老年代垃圾回收消耗时间
GCT：垃圾回收消耗总时间
```

12. jstat -printcompilation<font color=gray>**（JVM编译方法统计。输出已被JIT编译的方法）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstat -printcompilation 24528 1s 2
Compiled  Size  Type Method
    5607     21    1 java/util/TreeSet <init>
    5607     21    1 java/util/TreeSet <init>
    
解释：
Compiled：最近编译方法的数量
Size：最近编译方法的字节码数量
Type：最近编译方法的编译类型。
Method：方法名标识。
```

###### jinfo：java配置信息工具
+ 意义

    可以<font color=red> **实时查看和调整虚拟机各项参数**</font>。

+ 命令格式

    <font color=red> **jinfo [option] pid**</font>
    
    其中，`option`可选参数有：
    

选项 | 作用
---|---
-flag name | 打印指定VM标志的值
-flag [+-]name  | 启用或禁用指定的VM标志
-flag name=value | 将指定的VM标志设置为给定的值
-flags  | 打印所有VM标识的值
-sysprops | 打印Java系统属性
no option | 打印上述两个，即-flags与-sysprops所有值

+ 实例
1. jinfo -flags <font color=gray>**（打印所有VM标识的值）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jinfo -flags 24528
Attaching to process ID 24528, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14
Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC 
Command line:  -Dspring.output.ansi.enabled=always -Didea.launcher.port=7532 -Didea.launcher.bin.path=/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
```

2. jinfo -flag <name> <font color=gray>**（打印指定VM标志的值）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jinfo -flag InitialHeapSize  24528
-XX:InitialHeapSize=268435456
```

3. jinfo -flag  [+|-]name <font color=gray>**（启用或禁用指定的VM标志）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jinfo -flag -PrintGC 25598

该命令表示去除虚拟机线程25598VM配置中的PrintGC配置
```
备注：

+ 如果是布尔类型的JVM参数: jinfo -flag [+|-]<name> PID，enable or disable the named VM flag
+ 如果是数字/字符串类型的JVM参数 jinfo -flag <name>=<value> PID，to set the named VM flag to the given value

其中，需要注意的是，并不是所有的VM标识都可以进行动态配置的，比如：

```
houlongdeMacBook-Pro:JavaPattern houlong$ jinfo -flag MetaspaceSize=6m 25598
Exception in thread "main" com.sun.tools.attach.AttachOperationFailedException: flag 'MetaspaceSize' cannot be changed

	at sun.tools.attach.BsdVirtualMachine.execute(BsdVirtualMachine.java:213)
	at sun.tools.attach.HotSpotVirtualMachine.executeCommand(HotSpotVirtualMachine.java:261)
	at sun.tools.attach.HotSpotVirtualMachine.setFlag(HotSpotVirtualMachine.java:234)
	at sun.tools.jinfo.JInfo.flag(JInfo.java:134)
	at sun.tools.jinfo.JInfo.main(JInfo.java:81)
```
那么哪些参数可以通过jinfo实时调整？官方文档有如下描述：

```
Flags marked as manageable are dynamically writeable through the 
JDK management interface (com.sun.management.HotSpotDiagnosticMXBean API) and also through JConsole.

即：标记为manageable的参数或者通过com.sun.management.HotSpotDiagnosticMXBean这个类的接口得到属性，可以进行动态配置
```

1. 标记为manageable的Flags

    + Linux环境：java -XX:+PrintFlagsInitial | grep  manageable
    + Window环境：java -XX:+PrintFlagsInitial | findstr manageable
    
```
houlongdeMacBook-Pro:JavaPattern houlong$ java -XX:+PrintFlagsInitial | grep manageable
     intx CMSAbortablePrecleanWaitMillis            = 100                                 {manageable}
     intx CMSTriggerInterval                        = -1                                  {manageable}
     intx CMSWaitDuration                           = 2000                                {manageable}
     bool HeapDumpAfterFullGC                       = false                               {manageable}
     bool HeapDumpBeforeFullGC                      = false                               {manageable}
     bool HeapDumpOnOutOfMemoryError                = false                               {manageable}
    ccstr HeapDumpPath                              =                                     {manageable}
    uintx MaxHeapFreeRatio                          = 70                                  {manageable}
    uintx MinHeapFreeRatio                          = 40                                  {manageable}
     bool PrintClassHistogram                       = false                               {manageable}
     bool PrintClassHistogramAfterFullGC            = false                               {manageable}
     bool PrintClassHistogramBeforeFullGC           = false                               {manageable}
     bool PrintConcurrentLocks                      = false                               {manageable}
     bool PrintGC                                   = false                               {manageable}
     bool PrintGCDateStamps                         = false                               {manageable}
     bool PrintGCDetails                            = false                               {manageable}
     bool PrintGCID                                 = false                               {manageable}
     bool PrintGCTimeStamps                         = false                               {manageable}

```

    
2. 通过[HotSpotDiagnosticMXBean API](https://docs.oracle.com/javase/8/docs/jre/api/management/extension/com/sun/management/HotSpotDiagnosticMXBean.html)

```java
public class DiagnosticOptionsTest {

    public static void main(String[] args) {
        HotSpotDiagnostic mxBean = new HotSpotDiagnostic();
        List<VMOption> diagnosticVMOptions = mxBean.getDiagnosticOptions();
        for (VMOption vmOption:diagnosticVMOptions){
            System.out.println(vmOption.getName() + " = " + vmOption.getValue());
        }
    }
}

//output
HeapDumpBeforeFullGC = false
HeapDumpAfterFullGC = false
HeapDumpOnOutOfMemoryError = true
HeapDumpPath = /Users/houlong/workplace/JavaPattern/headDump.log
CMSAbortablePrecleanWaitMillis = 100
CMSWaitDuration = 2000
CMSTriggerInterval = -1
PrintGC = true
PrintGCDetails = false
PrintGCDateStamps = false
PrintGCTimeStamps = true
PrintGCID = false
PrintClassHistogramBeforeFullGC = false
PrintClassHistogramAfterFullGC = false
PrintClassHistogram = false
MinHeapFreeRatio = 0
MaxHeapFreeRatio = 100
PrintConcurrentLocks = false
UnlockCommercialFeatures = false
```

###### jmap：java内存映像工具
+ 意义

    用于<font color=red>**生成堆转储快照（一般称为heapdump或dump文件）**</font>。

+ 命令格式

    <font color=red>**jmap [option] <pid>**</font>
    
    其中，`option`可选参数有：
    

选项 | 意义
---|---
-dump | <font color=red>**生成java堆转储快照**</font>。格式：-dump:[live,]format=b,file=<filename> 其中live子参数说明是否只dump出存活的对象
-finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。
-heap | <font color=red>**显示java堆详细信息**</font>，如使用哪种回收器，参数配置，分代状况等
-histo[:live] | <font color=red>**显示堆中对象统计信息**</font>。包括类，实例数量，合计容量。live子参数说明是否只统计出存活的对象
-clstats | 打印类加载器统计信息
 -F | 当虚拟机进程对-dump选项或-histo没有响应时，可使用这个选项强制生成dump快照。其中，不支持子参数live。
 
 + 实例
1. jmap -dump<font color=gray>**（生成java堆转储快照）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jmap -dump:format=b,file=dump.bin 25772
Dumping heap to /Users/houlong/workplace/JavaPattern/dump.bin ...
Heap dump file created
```

2. jmap -finalizerinfo<font color=gray>**（显示在F-Queue中等待Finalizer线程执行finalize方法的对象）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jmap -finalizerinfo 25772
Attaching to process ID 25772, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14
Number of objects pending for finalization: 0

```

3. jmap -heap<font color=gray>**（显示java堆详细信息）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jmap -heap 25772
Attaching to process ID 25772, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 50
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 4294967296 (4096.0MB)
   NewSize                  = 89128960 (85.0MB)
   MaxNewSize               = 1431306240 (1365.0MB)
   OldSize                  = 179306496 (171.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 5242880 (5.0MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 5242880 (5.0MB)
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 67108864 (64.0MB)
   used     = 5369144 (5.120414733886719MB)
   free     = 61739720 (58.87958526611328MB)
   8.000648021697998% used
From Space:
   capacity = 11010048 (10.5MB)
   used     = 0 (0.0MB)
   free     = 11010048 (10.5MB)
   0.0% used
To Space:
   capacity = 11010048 (10.5MB)
   used     = 0 (0.0MB)
   free     = 11010048 (10.5MB)
   0.0% used
PS Old Generation
   capacity = 179306496 (171.0MB)
   used     = 0 (0.0MB)
   free     = 179306496 (171.0MB)
   0.0% used

879 interned Strings occupying 59648 bytes.
```

4. jmap -histo<font color=gray>**（显示堆中对象信息）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jmap -histo 25772

 num     #instances         #bytes  class name
----------------------------------------------
   1:           575        4214672  [I
   2:          4140         501720  [C
   3:          1097         272552  [B
   4:          2994          71856  java.lang.String
   5:           612          69528  java.lang.Class
   6:           673          45984  [Ljava.lang.Object;
   7:           721          23072  sun.management.Flag
   8:           152          10944  java.lang.reflect.Field
   9:           261           8352  java.io.File
  10:           334           8016  java.lang.Long
  11:           321           7704  java.lang.StringBuilder
  12:           113           7232  java.net.URL
```

4. jmap -clstats<font color=gray>**（打印类加载器统计信息）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jmap -clstats 25772
Attaching to process ID 25772, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.......done.
class_loader	classes	bytes	parent_loader	alive?	type

<bootstrap>	532	997889	  null  	live	<internal>
0x000000076ab3ffa0	0	0	  null  	live	sun/misc/Launcher$ExtClassLoader@0x00000007c000fa48
0x000000076ab6d738	35	109079	0x000000076ab3ffa0	live	sun/misc/Launcher$AppClassLoader@0x00000007c000f6a0

total = 3	567	1106968	    N/A    	alive=3, dead=0	    N/A    

解释：
从上面的输出中，我们可以发现：java程序大致都会使用3种类加载器：
1. 启动类加载器（Bootstrap ClassLoader）
2. 扩展类加载器（Extension ClassLoader）
3. 应用程序类加载器（Application ClassLoader）

且类加载器之间的层次关系为：双亲委派模型
```
![双亲委派模型](data:image/png;base64,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)


###### jhat：虚拟机堆转储快照分析工具

+ 意义

    该命令与jmap搭配使用，<font color=red>**用来分析jmap生成的堆转储快照**</font>。jhat内置了一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在浏览器中查看。
    
+ 实例

1. jhat dump.bin<font color=gray>**（分析jmap生成的dump文件）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jhat dump.bin 
Reading from dump.bin...
Dump file created Sun Apr 01 11:21:35 CST 2018
Snapshot read, resolving...
Resolving 15487 objects...
Chasing references, expect 3 dots...
Eliminating duplicate references...
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.

//备注：
其中，dump.bin文件是在介绍jmap命令时，执行命令：
jmap -dump:format=b,file=dump.bin 后生成的dump文件。
在使用jhat命令，显示“Server is ready”后，可以在浏览器中访问：http://localhost:7000/ 就可以查看分析结果
```

###### jstack：java堆栈跟踪工具

+ 意义

    该命令用于<font color=red>**生成虚拟机当前时刻的线程快照**</font>。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。生成线程快照的主要目的是<font color=red>
    **定位线程出现长时间停顿的原因，如线程间死锁，死循环等都是导致线程长时间停顿的常见原因。**</font>
    
+ 命令格式

    <font color=red>**jstack [option] vmid**</font>
    
    其中，`option`的可选参数有：
    

选项 | 作用
---|---
-F | 强制线程转储。 在jstack <pid>不响应时使用（进程挂起）
-l | 除堆栈外，显示关于锁的附加信息
-m | 如果调用到本地方法的话，可以显示C/C++的堆栈

+ 实例

1. jstack -l  24528<font color=gray>**（除堆栈外，额外显示锁信息）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstack -l  24528
2018-04-01 21:12:02
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.111-b14 mixed mode):

"Attach Listener" #72 daemon prio=9 os_prio=31 tid=0x00007f9eb629e800 nid=0x4d17 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"DestroyJavaVM" #71 prio=5 os_prio=31 tid=0x00007f9eb6b35000 nid=0x1403 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"http-nio-8000-AsyncTimeout" #69 daemon prio=5 os_prio=31 tid=0x00007f9eba3a2000 nid=0xb503 waiting on condition [0x000070001228d000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at org.apache.coyote.AbstractProtocol$AsyncTimeout.run(AbstractProtocol.java:1133)
	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
	- None
	
"VM Thread" os_prio=31 tid=0x00007f9eb6818000 nid=0x2e03 runnable 

"GC task thread#0 (ParallelGC)" os_prio=31 tid=0x00007f9eb601a000 nid=0x2607 runnable 
```

2. jstack -F<font color=gray>**（输出线程堆栈）**</font>

```
houlongdeMacBook-Pro:JavaPattern houlong$ jstack -F 25772
Attaching to process ID 25772, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14
Deadlock Detection:

No deadlocks found.

Thread 4871: (state = BLOCKED)


Thread 19715: (state = IN_NATIVE)
 - java.net.PlainSocketImpl.socketAccept(java.net.SocketImpl) @bci=0 (Interpreted frame)
 - java.net.AbstractPlainSocketImpl.accept(java.net.SocketImpl) @bci=7, line=409 (Interpreted frame)
 - java.net.ServerSocket.implAccept(java.net.Socket) @bci=60, line=545 (Interpreted frame)
 - java.net.ServerSocket.accept() @bci=48, line=513 (Interpreted frame)
 - com.intellij.rt.execution.application.AppMain$1.run() @bci=13, line=79 (Interpreted frame)
 - java.lang.Thread.run() @bci=11, line=745 (Interpreted frame)

Thread 3587: (state = IN_JAVA)
 - com.houlong.java.gc.StackOverFlowTest.main(java.lang.String[]) @bci=144, line=52 (Interpreted frame)
 - sun.reflect.NativeMethodAccessorImpl.invoke0(java.lang.reflect.Method, java.lang.Object, java.lang.Object[]) @bci=0 (Interpreted frame)
 - sun.reflect.NativeMethodAccessorImpl.invoke(java.lang.Object, java.lang.Object[]) @bci=100, line=62 (Interpreted frame)
 - sun.reflect.DelegatingMethodAccessorImpl.invoke(java.lang.Object, java.lang.Object[]) @bci=6, line=43 (Interpreted frame)
 - java.lang.reflect.Method.invoke(java.lang.Object, java.lang.Object[]) @bci=56, line=498 (Interpreted frame)
 - com.intellij.rt.execution.application.AppMain.main(java.lang.String[]) @bci=170, line=147 (Interpreted frame)

```



#### 参考文章
[jinfo命令详解](https://www.jianshu.com/p/c321d0808a1b)

[java高分局之jstat命令使用](https://blog.csdn.net/maosijunzi/article/details/46049117)