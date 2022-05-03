### 资料

JDK提供的监控工具入门看java guide的这篇文章：[JDK 监控和故障处理工具总结](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/JDK监控和故障处理工具总结?id=jdk-监控和故障处理工具总结)

具体每个JDK监控工具怎么使用，每个参数的意义，每一个打印的意义最好还是直接看Oracle的文档来的直接精准，比如jstat就可以看Oracle的[这篇文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html),其他排查工具比如jstack、jmap、jhat、jconsole工具的文档，都可以在Oracle的这个unix tools页面上找到入口：[Oracle-Unix-Tools-Page](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/)（网上文章一大抄，质量参差不齐，如果想要知道更多的细节和深层的东西，还是要看官方文档！）



# 常用命令

top (top -Hp pid (-H表示查看指定进程内的线程对系统资源占用的分析))

## jps 查看虚拟机进程状况

常用参数说明

**-m 输出传递给main方法的参数，如果是内嵌的JVM则输出为null。**

**-l 输出应用程序主类的完整包名，或者是应用程序JAR文件的完整路径。**

**-v 输出传给JVM的参数。**

## jstat	虚拟机统计信息监控工具

### 类加载统计：

```mipsasm
C:\Users\Administrator>jstat -class 2060
Loaded  Bytes  Unloaded  Bytes     Time
 15756 17355.6        0     0.0      11.29
```

- Loaded:加载class的数量
- Bytes：所占用空间大小
- Unloaded：未加载数量
- Bytes:未加载占用空间
- Time：时间

### 编译统计

```yaml
C:\Users\Administrator>jstat -compiler 2060
Compiled Failed Invalid   Time   FailedType FailedMethod
    9142      1       0     5.01          1 org/apache/felix/resolver/ResolverImpl mergeCandidatePackages
```

- Compiled：编译数量。
- Failed：失败数量
- Invalid：不可用数量
- Time：时间
- FailedType：失败类型
- FailedMethod：失败的方法

### 垃圾回收统计

```makefile
C:\Users\Administrator>jstat -gc 2060
 S0C    S1C    S0U    S1U      EC       EU        OC         OU          MC     MU    CCSC      CCSU   YGC     YGCT    FGC    FGCT     GCT
20480.0 20480.0  0.0   13115.3 163840.0 113334.2  614400.0   436045.7  63872.0 61266.5  0.0    0.0      149    3.440   8      0.295    3.735
```

- S0C：第一个幸存区的大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- OC：老年代大小
- OU：老年代使用大小
- MC：方法区大小
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

### 堆内存统计

```makefile
C:\Users\Administrator>jstat -gccapacity 2060
 NGCMN    NGCMX     NGC     S0C     S1C       EC      OGCMN      OGCMX       OGC         OC          MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC
204800.0 204800.0 204800.0 20480.0 20480.0 163840.0   614400.0   614400.0   614400.0   614400.0      0.0    63872.0  63872.0      0.0      0.0      0.0    149     8
```

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0C：第一个幸存区大小
- S1C：第二个幸存区的大小
- EC：伊甸园区的大小
- OGCMN：老年代最小容量
- OGCMX：老年代最大容量
- OGC：当前老年代大小
- OC:当前老年代大小
- MCMN:最小元数据容量
- MCMX：最大元数据容量
- MC：当前元数据空间大小
- CCSMN：最小压缩类空间大小
- CCSMX：最大压缩类空间大小
- CCSC：当前压缩类空间大小
- YGC：年轻代gc次数
- FGC：老年代GC次数

### 新生代垃圾回收统计

```makefile
C:\Users\Administrator>jstat -gcnew 7172
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
40960.0 40960.0 25443.1    0.0 15  15 20480.0 327680.0 222697.8     12    0.736
```

- S0C：第一个幸存区大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- TT:对象在新生代存活的次数
- MTT:对象在新生代存活的最大次数
- DSS:期望的幸存区大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间

### 新生代内存统计

```makefile
C:\Users\Administrator>jstat -gcnewcapacity 7172
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
  409600.0   409600.0   409600.0  40960.0  40960.0  40960.0  40960.0   327680.0   327680.0    12     0
```

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0CMX：最大幸存1区大小
- S0C：当前幸存1区大小
- S1CMX：最大幸存2区大小
- S1C：当前幸存2区大小
- ECMX：最大伊甸园区大小
- EC：当前伊甸园区大小
- YGC：年轻代垃圾回收次数
- FGC：老年代回收次数

### 老年代垃圾回收统计

```makefile
C:\Users\Administrator>jstat -gcold 7172
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
 33152.0  31720.8      0.0      0.0    638976.0    184173.0     12     0    0.000    0.736
```

- MC：方法区大小
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- OC：老年代大小
- OU：老年代使用大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

### 老年代内存统计

```makefile
C:\Users\Administrator>jstat -gcoldcapacity 7172
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
   638976.0    638976.0    638976.0    638976.0    12     0    0.000    0.736
```

- OGCMN：老年代最小容量
- OGCMX：老年代最大容量
- OGC：当前老年代大小
- OC：老年代大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

### 元数据空间统计

```makefile
C:\Users\Administrator>jstat -gcmetacapacity 7172
   MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT
   0.0    33152.0    33152.0        0.0        0.0        0.0    12     0    0.000    0.736
```

- MCMN:最小元数据容量
- MCMX：最大元数据容量
- MC：当前元数据空间大小
- CCSMN：最小压缩类空间大小
- CCSMX：最大压缩类空间大小
- CCSC：当前压缩类空间大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

### 总结垃圾回收统计

```mipsasm
C:\Users\Administrator>jstat -gcutil 7172
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 62.12   0.00  81.36  28.82  95.68      -     12    0.736     0    0.000    0.736
```

- S0：幸存1区当前使用比例
- S1：幸存2区当前使用比例
- E：伊甸园区使用比例
- O：老年代使用比例
- M：元数据区使用比例
- CCS：压缩使用比例
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

## jstack	java堆跟踪工具

jmap	java内存印象工具

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/deploy/logs/dump

**其实最好是在服务器启动的时候加上VM参数  -XX:+HeapDumpOnOutOfMemoryError  可以让虚拟机在 OOM 异常出现之后自动生成 dump 文件，Linux 命令下可以通过 `kill -3` 发送进程退出信号也能拿到 dump 文件**



### 命令使用的一个实际案例

服务器CPU突然高负载:

1. 使用top命令，查看进程CPU使用情况，top命令默认按照CPU使用率来排序（进入top的程序后，输入M可以以内存使用量排序，P以CPU使用量排序）。

2. 发现PID=80421，COMMAND=java的进程占用了200%的”%CPU“。

3. 使用 jps -lv | grep 80421,查看这个java进程的详细信息。

4. 使用top -Hp 80421命令，-p指定进程， -H 显示指定进程下的线程对服务器资源的使用情况.

5. 查看到进程80421中线程id=80637的线程占用了大量的cpu.

6. 线程id=80637，它的16进制为：13afd（[一个在线数字进制转换的工具](https://www.sojson.com/hexconvert/10to16.html)）

7. jstack 80421 | less，使用jstack命令来查看我们这个JAVA进程的虚拟机栈情况，因为线程数可能会很多，全部都打印出来太乱了，所以使用linux的管道符”|“，将jstack的输出交给less命令来管理，使用less命令就能很方便的浏览jstack 80421 命令执行之后的输出信息了。

8. 在jstack 80421 | less中，搜索那个占用大量CPU的线程id ：/13afd。搜索到”nid=0x13afd“（nid即native thread id）的线程的调用栈，查看代码，发现线程一直在执行这个while循环，while循环判断某个信号值，为true就一直执行，所以是因为这个信号值没有被更新成false，才导致了死循环，接下来就是排查我们代码的问题了。（线上排查代码可以使用阿里的arthas工具）

9. ```java
   查看前20大的对象
   jmap -histo ${pid}|head -20
   ```

   可以查看占用空间的大对象

10. ```
      jmap -dump:format=b,file=C:\Users\SnailClimb\Desktop\heap.hprof 17340
    ```

11. jstack -info ${pid}  查看当前启动参数

12. jstat -gc ${pid} 500   每隔500毫秒输出GC信息







### arthas工具常用命令

1. dashboard 图形界面，默认5s刷新一次