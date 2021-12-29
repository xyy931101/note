# JVM

## JVM内存模型（运行时数据区）

​	JVM中，将内存分为PC寄存器、本地方法栈、虚拟机栈、堆、及元空间(也叫方法区)

### PC寄存器（Program Counter Register）

​	PC寄存器是用来存储指向下一条指令的地址，也即将将要执行的指令代码。由执行引擎读取下一条指令。

​	1.它是一块很小的内存空间，几乎可以忽略不计。也是运行速度最快的存储区域

​	2.在jvm规范中，每个线程都有它自己的程序计数器，是线程私有的，生命周期与线程的生命周期保持一致

​	3.任何时间一个线程都只有一个方法在执行，也就是所谓的当前方法。程序计数器会存储当前线程正在执行的java方法的JVM指令地	      	址；或者，如果实在执行native方法，则是未指定值（undefined）,因为程序计数器不负责本地方法栈。

​	4.它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成

​	5.字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令

​	6.它是唯一一个在java虚拟机规范中没有规定任何OOM（Out Of Memery）情况的区域,而且没有垃圾回收

### 虚拟机栈

栈：可以使用数组或链表来实现。

> 操作数栈：每个独立的栈帧中处理包含局部标量表以外还包含一个后进先出的**操作数栈**，也可以称为**表达式栈**。操作数栈，在执行方法过程中，根据字节码指令，往栈中写入数据或提取数据，即入栈(push)/出栈(pop)。
>
> 如果当前方法有返回的话，其返回值将会压入当前栈帧的操作数栈中，并更新PC寄存器中的下一条需要执行的字节码指令。
>
> 其中需要注意的是，long,duble是占两个数组的位置的   **StackOverflowError**

### 本地方法栈

> 和Java虚拟机栈很类似，不同的是本地方法栈为Native方法服务。  **StackOverflowError**

### Java堆

> 是Java虚拟机所管理的内存中最大的一块。由所有线程共享，在虚拟机启动时创建。堆区唯一目的就是存放对象实例。
>
>   堆中可细分为新生代和老年代，再细分可分为Eden空间、From Survivor空间、To Survivor空间。
>
>   堆无法扩展时，抛出OutOfMemoryError异常  **Java Heap space**

### 方法区（Method Area）也叫永久代(1.7)

> 所有线程共享，存储内容：类型信息、常量、静态变量、及时编译器编译后的代码缓存。
>
>   当方法区无法满足内存分配需求时，抛出OutOfMemoryError **PermGen space**

### 运行时常量池

> 它是方法区的一部分，Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项是常量池（Const Pool Table），用于存放编译期生成的各种字面量和符号引用。并非预置入Class文件中常量池的内容才进入方法运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法。
>
>   当方法区无法满足内存分配需求时，抛出OutOfMemoryError   **PermGen space**

### 直接内存

> 并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。
>
> JDK1.4加入了NIO，引入一种基于通道与缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。因为避免了在Java堆和Native堆中来回复制数据，提高了性能。
>
> 当各个内存区域总和大于物理内存限制，抛出OutOfMemoryError异常。

## 对象的实例化内存布局与访问定位

### 	创建对象的方式

1. new 关键字
2. class.newInsatnce			
3. Contructor的newInstance
4. 使用clone
5. 使用反序列化
6. 第三方库：Objenesjs

### 创建对象的步骤

1. 判断对象的类是否有加载、连接、初始化(这个过程可以判断对象的大小)

2. 为对象开辟内存空间

   内存分配方式:**指针碰撞**(仅仅是把指针往空闲的地方挪动与对象大小的相等的距离)

   **TLAB**(对象分配过程，Thread Local Allocation Buffer) 

   即：每个线程预先分配一块小内存，哪个线程要分配内存就在哪个线程的缓冲区进行分配	

   - 从内存模型而不是垃圾回收的角度，对Eden区域据继续进行划分，**JVM为每个线程分配了一个私有的缓存区域**，他包含在Eden区域

   - 多线程同时分配内存时，使用TLAB可以避免一系列的非线程安全问题，同时还能提升内存分配的吞吐量，因此我们称这种内存分配为**快速分配策略**

3. 初始化分配的空间--所有属性设置默认值，保证对象不赋值时可以直接使用

4. 设置对象的对象头

5. 显示初始化（代码块中初始化、构造器中初始化）

![image-20210124221734023](image\image-20210124221734023.png)

### 对象的内存布局

#### 对象头

- 运行时元数据MarkWord
  1. 哈希值 hascode
  2. GC分代年龄
  3. 锁状态标志
  4. 线程持有的锁
  5. 偏向线程ID
  6. 偏向时间戳
- 类型指针：  指向类元数据InstanceClass，确定该对象的类型
- 如果是数组，则还需要记录数组的长度

#### 示例数据（Instance Data）

- 他是对象真正存储的有效信息，包括程序代码中定义的各种类型的字段（包括从父类继承的及本身的字段）

- 规则
  1. 相同宽度的字段总数被分配在一起
  2. 父类中定义的变量会出现在自身变量之前
  3. 如果CompactField参数为true，子类的窄变量可能会插入到父类变量的空隙

#### 对齐填充（Padding）

不是必须的，也没特别含义，仅仅起到占位符作用

### 对象的访问定位

目前主流方式主要有两种：

- 划出一块内存作为句柄池，reference中存储这对象的访问句柄，而句柄中包含了对象的实例数据与类型数据。
- 直接指针：refrence存储这直接对象地址，然后对象再存储着类型指针。

HotSpot主要使用第二种方式，因为JAVA访问对象非常频繁，可以减少一次类型指针的访问开销



### JVM参数设置

- **堆大小设置**

> ​	-Xms:用来设置堆空间(新生代+老年代)的**初始**内存大小
>
> ​		-X: 是JVM的运行参数
>
> ​		ms:memory start 初始内存
>
> ​	-Xmx:用来设置堆空间(新生代+老年代)的**最大**内存大小
>
> ​		mx:memory max 最大内存

- **默认设置**

  > 初始化内存大小：电脑内存大小/64

>最大内存大小：电脑内存/4
>
>新生代与老年代比例 默认1:2，也就是新生代占整个内存的1/3

- **手动设置**

  -Xms600m -Xmx600m -XX:NewRatio=4

  开发中建议将初始堆内存和最大内存设置成相同的值 避免频繁FullGC

- **查看参数设置**

  jps 查看进行ID jstat -gc ${pid}查看进程各个区的大小

## GC(垃圾回收器)

### 如何定位垃圾

1. 引用计数法

2. 可达性分析算法（Reachability Analysis）

   GC roots:**线程变量、静态变量、常量池、JNI引用对象(即Native方法)、class对象**

### 常见的垃圾收集算法

- Mark-Sweep (标记清除)

  > 对需要回收的对象进行标记，然后再进行清除
  >
  > 缺点：
  >
  > 1. 执行效率不稳定，因为大量对象都是进行回收的，所以需要进行大量标记和清除动作，导致标记和清除动作效率随着对象增多而降低
  > 2. 内存碎片化，空间不连续

- Semispace Coping(拷贝)

  > 将活着的对象复制到另外一个空间上面去，从而回收整个下半区。实现简单，运行效率也比较高
  >
  > 缺点：比较浪费空间，老年代不能采用这个算法
  
- Mark-Compact(标记整理) 

  > 对所有活着的对象进行标记，然后向内存的另外一端移动，之后对边界外的空间进行整理
  >
  > 缺点：移动存活的对象必须要STW(Stop The Word),停顿时间更长。

- 三色标记算法

  > - 白色：没有检查
  >
  > - 灰色：自身被检查了，成员没被检查完（可以认为访问到了，但是正在被检查，就是图的遍历里那些在队列中的节点）
  >
  > - 黑色：自身和成员都被检查完了
  >
  >   
  >
  >   产生漏标： 1. 标记进行时增加了一个黑到白的引用，如果不重新对黑色进行处理，则会漏标 
  >
  >   					2. 标记进行时删除了灰对象到白对象的引用，那么这个白对象有可能被漏标

  打破上述两个条件之一即可 

  1. incremental update -- 增量更新，关注引用的增加， 把黑色重新标记为灰色，下次重新扫描属性，CMS使用 
  2. SATB snapshot at the beginning – 关注引用的删除 当B->D消失时，要把这个引用推到GC的堆栈，保证D还能被GC扫描到 G1使用

### JVM分代模型

1. 部分垃圾回收器使用模型
2. 新生代、老年代、永久代(1.7)/元数据区(1.8) Metaspace

   > 1. 永久代数据  --Class
   > 2. 永久代必须制定大小限制、元数据区可以设置也可以不设置（受限于物理内存）
   > 3. 字符串常量1.7-永久代、1.8元数据
   > 4. MeathArea逻辑概念，永久区、元数据区
3. 新生代 =Eden+2个suvivor区 

   > 1. YGC之后，大多数对象会被回收，活着的对象进入suvivor0
   > 2. 再次YGC、活着的对象eden+s0->s1
   > 3. 再次YGC、活着对对象eden+s1->s0
   > 4. 年龄足够->老年代
4. 老年代

   > 1. 顽固分子
   > 2. 老年代满了  Major GC
5. GC Tuning(Generation)

   > 1. 尽量减少FGC
   > 2. MinorGC = YGC
   > 3. MajorGC = FGC

**其中：跨代引用是有新生代的记忆集（Remembered Set）来进行标识，Minor GC的时候，这部分也会被加入的GCRoots里面**

### 内存分配与回收策略

针对不同年龄段的对象分配原则如下

- 优先分配到Eden

- 大对象直接分配到老年代

  尽量避免出现过多的大对象

- 动态对象年龄判断

  如果Survivor区中相同年龄的所有对象大小总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无需等到MaxTenuringThreshold中年龄的要求

- 空间分配担保：

  - -XX:MaxTenuringThreshold

### 常见的垃圾回收器

#### serial

> 年轻代串行回收，只有一个线程，性能比较低

#### PS（Parallel Scavenge）

> 并行回收---**多线程**，与ParNew 区别是：主要关注吞吐量,即：**减少运行垃圾收集的时间**

#### ParNew 

> 年轻代配合CMS的并行回收

#### SerialOld  

> 老年代串行回收，与serial搭配使用

#### ParallelOld

>  PS的老年代收集器。

#### CMS(Concurrent Mark Sweep) 

> 老年代，并发的。垃圾回收和应用程序同时运行，降低STW的时间。其一般跟**ParNew**一起搭配使用，当其预留空间放不下用户进程分配对象的需求的话，则不得不冻结所有用户进程，启用**Serial Old**收集器进行老年代的垃圾回收

##### CMS步骤：

1. **初始标记**（CMS initial mark）

   仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿（STW -Stop the world）

2. **并发标记（**CMS concurrent mark）

   从 GC Root 开始对堆中对象进行可达性分析，找到存活对象，它在整个回收过程中耗时最长，不需要停顿

3. **重新标记**（CMS remark）

   为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿(STW)。这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短

4. **并发清除**（CMS concurrent sweep）不需要停顿

![](image\CMS垃圾回收过程.jpg)

##### CMS明显的缺点：

> - CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。在JDK1.5的默认设置下，CMS收集器当老年代使用了68%的空间后就会被激活；
>
>
> - CMS是基于“标记-清除”算法实现的收集器，手机结束时会有大量空间碎片产生。空间碎片过多，可能会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前出发FullGC。



#### G1(10ms) 三色标记 + SATB

**逻辑分代，物理不分代**

> ▪ 并发收集 
>
> ▪ 压缩空闲空间不会延长GC的暂停时间； 
>
> ▪ 更易预测的GC暂停时间； 
>
> ▪ 适用不需要实现很高的吞吐量的场景

[可能是最全面的G1学习笔记]: https://zhuanlan.zhihu.com/p/54048685

#### ZGC(1ms)

#### Shenandoah

### 如果采用ParNew + CMS组合，怎样做才能够让系统基本不产生FGC

> 1.加大JVM内存
>
> 2.加大Young的比例
>
> 3.提高Y-O的年龄
>
> 4.提高S区比例
>
> 5.避免代码内存泄漏

### GC常用参数

> - -Xmn -Xms -Xmx -Xss 年轻代 最小堆 最大堆 栈空间
> - -XX:+UseTLAB 使用TLAB，默认打开
> - -XX:+PrintTLAB 打印TLAB的使用情况
> - -XX:TLABSize 设置TLAB大小
> - -XX:+DisableExplictGC System.gc()不管用 ，FGC
> - -XX:+PrintGC
> - -XX:+PrintGCDetails
> - -XX:+PrintHeapAtGC
> - -XX:+PrintGCTimeStamps
> - -XX:+PrintGCApplicationConcurrentTime (低) 打印应用程序时间
> - -XX:+PrintGCApplicationStoppedTime （低） 打印暂停时长
> - -XX:+PrintReferenceGC （重要性低） 记录回收了多少种不同引用类型的引用
> - -verbose:class 类加载详细过程
> - -XX:+PrintVMOptions
> - -XX:+PrintFlagsFinal  -XX:+PrintFlagsInitial 必须会用
> - -Xloggc:opt/log/gc.log
> - -XX:MaxTenuringThreshold 升代年龄，最大值15
> - 锁自旋次数 -XX:PreBlockSpin 热点代码检测参数-XX:CompileThreshold 逃逸分析 标量替换 ... 这些不建议设置

### Parallel常用参数

> - -XX:SurvivorRatio
> - -XX:PreTenureSizeThreshold 大对象到底多大
> - -XX:MaxTenuringThreshold
> - -XX:+ParallelGCThreads 并行收集器的线程数，同样适用于CMS，一般设为和CPU核数相同
> - -XX:+UseAdaptiveSizePolicy 自动选择各区大小比例

### CMS常用参数

> - -XX:+UseConcMarkSweepGC
> - -XX:ParallelCMSThreads CMS线程数量
> - -XX:CMSInitiatingOccupancyFraction 使用多少比例的老年代后开始CMS收集，默认是68%(近似值)，如果频繁发生SerialOld卡顿，应该调小，（频繁CMS回收）
> - -XX:+UseCMSCompactAtFullCollection 在FGC时进行压缩
> - -XX:CMSFullGCsBeforeCompaction 多少次FGC之后进行压缩
> - -XX:+CMSClassUnloadingEnabled
> - -XX:CMSInitiatingPermOccupancyFraction 达到什么比例时进行Perm回收
> - GCTimeRatio 设置GC时间占用程序运行时间的百分比
> - -XX:MaxGCPauseMillis 停顿时间，是一个建议时间，GC会尝试用各种手段达到这个时间，比如减小年轻代

### G1常用参数

> - -XX:+UseG1GC
> - -XX:MaxGCPauseMillis 建议值，G1会尝试调整Young区的块数来达到这个值
> - -XX:GCPauseIntervalMillis ？GC的间隔时间
> - -XX:+G1HeapRegionSize 分区大小，建议逐渐增大该值，1 2 4 8 16 32。 随着size增加，垃圾的存活时间更长，GC间隔更长，但每次GC的时间也会更长 ZGC做了改进（动态区块大小）
> - G1NewSizePercent 新生代最小比例，默认为5%
> - G1MaxNewSizePercent 新生代最大比例，默认为60%
> - GCTimeRatio GC时间建议比例，G1会根据这个值调整堆空间
> - ConcGCThreads 线程数量
> - InitiatingHeapOccupancyPercent 启动G1的堆空间占用比例

------

## 类文件结构

### 魔法数字（MagicNumber）

​		0xCAFEBABE:确定是一个class文件，第4个字节是class文件的版本号，**第5和第6个字节是次版本号，第7和第8个字节是主版本号**，JAVA的版本号是从45开始的

### 常量池

> - 被模块导出或开放的包
> - 类和接口的全限定名
> - 字段的名称和描述符
> - 方法的名称和描述符
> - 方法句柄和方法类型
> - 动态调用点和动态常量

### 访问标志

> 用于标识类或接口的访问信息

### 类索引，父索引与接口索引集合

### 字段表集合

### 方法表集合

### 属性表集合

### InnerClass属性

### Deprecated及Synthetic属性

### StackMapTable属性

### Signature属性



------

## 类加载机制

### JVM类加载过程

​		类的加载过程可以分为： 加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序进行，而解析阶段则不一定，它在某些情况下可能在初始化阶段后在开始，因为java支持运行时绑定。

#### 1.加载（loading）阶段

```html
		简单的说，类加载阶段就是由类加载器负责根据一个类的全限定名来读取此类的二进制字节流到JVM内部，并存储在运行时内存区的方法区，然后将其转换为一个与目标类型对应的java.lang.Class对象实例（Java虚拟机规范并没有明确要求一定要存储在堆区中，只是hotspot选择将Class对戏那个存储在方法区中），这个Class对象在日后就会作为方法区中该类的各种数据的访问入口。
```

#### 2.链接（linking）阶段

```
1）、验证
		验证类是否符合JVM规范，是否是一个有效的字节码文件，验证内容涵盖了类数据信息的格式验证、语义分析、操作验证等。
- 格式验证：验证是否符合class文件规范
- 语义验证：检查一个被标记为final的类型是否包含子类；检查一个类中的final方法视频被子类进行重写；确保父类和子类之间没有不兼容的一些方法声明（比如方法签名相同，但方法的返回值不同）
- 操作验证：在操作数栈中的数据必须进行正确的操作，对常量池中的各种符号引用执行验证（通常在解析阶段执行，检查是否通过富豪引用中描述的全限定名定位到指定类型上，以及类成员信息的访问修饰符是否允许访问等）
- 2）、准备
  		为类中的所有静态变量分配内存空间，并为其设置一个初始值（由于还没有产生对象，实例变量不在此操作范围内）被final修饰的静态变量，会直接赋予原值；类字段的字段属性表中存在ConstantValue属性，则在准备阶段，其值就是ConstantValue的值
  3）、解析
  		将常量池中的符号引用转为直接引用（得到类或者字段、方法在内存中的指针或者偏移量，以便直接调用该方法），这个可以在初始化之后再执行。可以认为是一些静态绑定的会被解析，动态绑定则只会在运行是进行解析；静态绑定包括一些final方法(不可以重写),static方法(只会属于当前类)，构造器(不会被重写)
```

#### 3.初始化（init）阶段

```
	将一个类中所有被static关键字标识的代码统一执行一遍，如果执行的是静态变量，那么就会使用用户指定的值覆盖之前在准备阶段设置的初始值；如果执行的是static代码块，那么在初始化阶段，JVM就会执行static代码块中定义的所有操作。
	所有类变量初始化语句和静态代码块都会在编译时被前端编译器放在收集器里头，存放到一个特殊的方法中，这个方法就是<clinit>方法，即类/接口初始化方法。该方法的作用就是初始化一个中的变量，使用用户指定的值覆盖之前在准备阶段里设定的初始值。任何invoke之类的字节码都无法调用<clinit>方法，因为该方法只能在类加载的过程中由JVM调用。
	如果父类还没有被初始化，那么优先对父类初始化，但在<clinit>方法内部不会显示调用父类的<clinit>方法，由JVM负责保证一个类的<clinit>方法执行之前，它的父类<clinit>方法已经被执行。	
	JVM必须确保一个类在初始化的过程中，如果是多线程需要同时初始化它，仅仅只能允许其中一个线程对其执行初始化操作，其余线程必须等待，只有在活动线程执行完对类的初始化操作之后，才会通知正在等待的其他线程。
```



### 类加载器

​	用户自定义类,是由用户自定义加载器进行加载的,java的核心类库，是由引导类加载器进行加载的。

```java
public static void main(String[] args) {

        //系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);

        //拓展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader);

        //bootStrap(引导类加载器)加载器,最顶层,获取不到
        ClassLoader bootStrap = extClassLoader.getParent();
        System.out.println(bootStrap);

        //对于用户自定义来说,使用系统类加载器进行加载
        ClassLoader classLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(classLoader);

        ClassLoader stringClassLoader = String.class.getClassLoader();
        System.out.println(stringClassLoader);
    }
得到结果为
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@6659c656
null
sun.misc.Launcher$AppClassLoader@18b4aac2
null
```

### 用户自定义类加载器

​	在日常开发中，用户基本上用引导类加载器、拓展类加载器，系统类加载器这3中互相配合使用的。在必要时，还可以自定义类的加载器，来定制类的加载方式。用户自定义类的加载器的时候，一般继承**URLClassLoader**，可以尽量减少用户的开发

- 隔离加载类
- 修改类的加载方式
- 拓展加载源
- 防止源代码泄露



------

## 强软弱虚引用

（1）强引用是使用最普遍的引用。**只要某个对象有强引用与之关联，JVM必定不会回收这个对象，即使在内存不足的情况下，JVM宁愿抛出OutOfMemory错误也不会回收这种对象**

（2）软引用是用来描述一些有用但并不是必需的对象，在Java中用java.lang.ref.SoftReference类来表示。**只有在内存不足的时候JVM才会回收该对象。**软引用可用来实现内存敏感的高速缓存。 软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中

（3）只具有弱引用的对象拥有**更短暂的生命周期**。在**GC**的过程中，一旦发现了**只具有弱引用的对象**，不管当前内存空间足够与否，都会**立即**回收它的内存。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

（4）虚引用也称为**幻影引用**，一个对象是都有虚引用的存在都不会对生存时间都构成影响，也无法通过虚引用来获取对一个对象的真实引用。唯一的用处：能在对象被GC时收到系统通知，JAVA中用PhantomReference来实现虚引用

虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中

## ThreaLocal

Java中**每一个线程都有自己的专属本地变量**， JDK 中提供的`ThreadLocal`类，**`ThreadLocal`类主要解决的就是让每个线程绑定自己的值，可以将`ThreadLocal`类形象的比喻成存放数据的盒子，盒子中可以存储每个线程的私有数据。**

1. ThreadLocal是Java中所提供的线程本地存储机制，可以利用该机制将数据存在某个线程内部，该线程可以在任意时刻、任意方法中获取缓存的数据
2. ThreadLocal底层是通过ThreadLocalmap来实现的，每个Thread对象（注意不是ThreadLocal对象）中都存在一个ThreadLocalMap，Map的key为ThreadLocal对象，Map的value为需要缓存的值WeakReference的Entry。
3. ThreadLocal经典的应用场景就是**Spring的Trancation**连接管理（一个线程持有一个链接，该连接对象可以在不同给的方法之间进行线程传递，线程之间不共享同一个连接）

### 用它可能会带来什么问题

如果在线程池中使用ThreadLocal会造成**内存泄漏**，因为当ThreadLocal对象使用完之后，应该要把设置的key，value，也就是Entry对象进行回收，但线程池中的线程不会回收，而线程对象是通过**强引用**指向ThreadLocalmap，ThreadLocalmap也是通过强引用指向Entry对象，线程不被回收，Entry对象也就不会被回收，从而出现内存泄漏，**解决办法是**：在使用了ThreadLocal对象之后，手动调用ThreadLocal的**remove**方法，手动清除Entry对象。
