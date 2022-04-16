### 资料

JDK提供的监控工具入门看java guide的这篇文章：[JDK 监控和故障处理工具总结](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/JDK监控和故障处理工具总结?id=jdk-监控和故障处理工具总结)

具体每个JDK监控工具怎么使用，每个参数的意义，每一个打印的意义最好还是直接看Oracle的文档来的直接精准，比如jstat就可以看Oracle的[这篇文档](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html),其他排查工具比如jstack、jmap、jhat、jconsole工具的文档，都可以在Oracle的这个unix tools页面上找到入口：[Oracle-Unix-Tools-Page](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/)（网上文章一大抄，质量参差不齐，如果想要知道更多的细节和深层的东西，还是要看官方文档！）



### 常用命令

top (top -Hp pid (-H表示查看指定进程内的线程对系统资源占用的分析))

jps 查看虚拟机进程状况

jstat	虚拟机统计信息监控工具

jstack	java堆跟踪工具

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