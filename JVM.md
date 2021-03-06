# JVM

## JVM类加载过程

​		类的加载过程可以分为： 加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序进行，而解析阶段则不一定，它在某些情况下可能在初始化阶段后在开始，因为java支持运行时绑定。

###### 1.**加载（loading）阶段**

​			简单的说，类加载阶段就是由类加载器负责根据一个类的全限定名来读取此类的二进制字节流到JVM内部，并存储在运行时内存区的方法区，然后将其转换为一个与目标类型对应的java.lang.Class对象实例（Java虚拟机规范并没有明确要求一定要存储在堆区中，只是hotspot选择将Class对戏那个存储在方法区中），这个Class对象在日后就会作为方法区中该类的各种数据的访问入口。

​	

###### 2.**链接（linking）阶段**

1）、验证
验证类数据信息是否符合JVM规范，是否是一个有效的字节码文件，验证内容涵盖了类数据信息的格式验证、语义分析、操作验证等。
格式验证：验证是否符合class文件规范
语义验证：检查一个被标记为final的类型是否包含子类；检查一个类中的final方法视频被子类进行重写；确保父类和子类之间没有不兼容的一些方法声明（比如方法签名相同，但方法的返回值不同）
操作验证：在操作数栈中的数据必须进行正确的操作，对常量池中的各种符号引用执行验证（通常在解析阶段执行，检查是否通过富豪引用中描述的全限定名定位到指定类型上，以及类成员信息的访问修饰符是否允许访问等）

2）、准备
为类中的所有静态变量分配内存空间，并为其设置一个初始值（由于还没有产生对象，实例变量不在此操作范围内）被final修饰的静态变量，会直接赋予原值；类字段的字段属性表中存在ConstantValue属性，则在准备阶段，其值就是ConstantValue的值
3）、解析
将常量池中的符号引用转为直接引用（得到类或者字段、方法在内存中的指针或者偏移量，以便直接调用该方法），这个可以在初始化之后再执行。可以认为是一些静态绑定的会被解析，动态绑定则只会在运行是进行解析；静态绑定包括一些final方法(不可以重写),static方法(只会属于当前类)，构造器(不会被重写)



###### 3.**初始化（init）阶段**

将一个类中所有被static关键字标识的代码统一执行一遍，如果执行的是静态变量，那么就会使用用户指定的值覆盖之前在准备阶段设置的初始值；如果执行的是static代码块，那么在初始化阶段，JVM就会执行static代码块中定义的所有操作。
所有类变量初始化语句和静态代码块都会在编译时被前端编译器放在收集器里头，存放到一个特殊的方法中，这个方法就是<clinit>方法，即类/接口初始化方法。该方法的作用就是初始化一个中的变量，使用用户指定的值覆盖之前在准备阶段里设定的初始值。任何invoke之类的字节码都无法调用<clinit>方法，因为该方法只能在类加载的过程中由JVM调用。
如果父类还没有被初始化，那么优先对父类初始化，但在<clinit>方法内部不会显示调用父类的<clinit>方法，由JVM负责保证一个类的<clinit>方法执行之前，它的父类<clinit>方法已经被执行。
JVM必须确保一个类在初始化的过程中，如果是多线程需要同时初始化它，仅仅只能允许其中一个线程对其执行初始化操作，其余线程必须等待，只有在活动线程执行完对类的初始化操作之后，才会通知正在等待的其他线程。

### 类加载器：

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



![image-20210118232801993](C:\Users\xiongyy\Desktop\技术资料\图片\image-20210118232801993.png)



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

### 虚拟机栈 局部变量表

栈：可以使用数组或链表来实现。

操作数栈：每个独立的栈帧中处理包含局部标量表以外还包含一个后进先出的**操作数栈**，也可以称为**表达式栈**。

​				  操作数栈，在执行方法过程中，根据字节码指令，往栈中写入数据或提取数据，即入栈(push)/出栈(pop)。

​				  如果当前方法有返回的话，其返回值将会压入当前栈帧的操作数栈中，并更新PC寄存器中的下一条需要执行的字节码指令。

​				  其中需要注意的是，long,duble是占两个数组的位置的。

### JVM参数设置

- **堆大小设置**

​	-Xms:用来设置堆空间(新生代+老年代)的**初始**内存大小

​		-X: 是JVM的运行参数

​		ms:memory start 初始内存

​	-Xmx:用来设置堆空间(新生代+老年代)的**最大**内存大小

​		mx:memory max 最大内存

​	-XX:NewRatio 新生代与老年代比例

​	-Xmn:设置新生代空间的大小(初始值及最大值)

​	-XX:+PrintGCDetails 打印详细的GC处理日志

​		-XX:PrintGc 打印GC简要日志

​	

- **默认设置**

  初始化内存大小：电脑内存大小/64

  最大内存大小：电脑内存/4

  新生代与老年代比例 默认1:2，也就是新生代占整个内存的1/3

- **手动设置**

  -Xms600m -Xmx600m -XX:NewRatio=4

  开发中建议将初始堆内存和最大内存设置成相同的值 避免频繁FullGC

- **查看参数设置**

  jps 查看进行ID jstat -gc ${pid}查看进程各个区的大小


### 垃圾回收（Minor GC,Major GC,Full GC）

​	JVM在进行GC时，并非每次都都对上面三个内存都进行回收(新生代，老年代，方法区)，大部分都是新生代进行回收的

​	HotSpot虚拟机按照回收区域又分两大种类型：一种是部分收集（Partial GC）,一种是整堆收集（Full GC）

- ​	部分收集：不是完整收集整个Java堆的垃圾收集，其中又分为：
  1. 新生代收集（MinorGC/YoungGC）：只是新生代的收集
  2. 老年代收集（MajorGC/OldGC）:实在老年代的垃圾收集
- 整堆收集（FullGC）：收集整个Java堆和方法区的回收



### 内存分配策略

针对不同年龄段的对象分配原则如下

- 优先分配到Eden

- 大对象直接分配到老年代

  尽量避免出现过多的大对象

- 动态对象年龄判断

  如果Survivor区中相同年龄的所有对象大小总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无需等到MaxTenuringThreshold中年龄的要求

- 空间分配担保：

  - -XX:

### TLAB(对象分配过程，Thread Local Allocation Buffer)

- 从内存模型而不是垃圾回收的角度，对Eden区域据继续进行划分，**JVM为每个线程分配了一个私有的缓存区域**，他包含在Eden区域

- 多线程同时分配内存时，使用TLAB可以避免一系列的非线程安全问题，同时还能提升内存分配的吞吐量，因此我们称这种内存分配为**快速分配策略**


### 方法区（Method Area）也叫永久代(1.7)

​	存储内容：类型信息、常量、静态变量、及时编译器编译后的代码缓存。

​	其中永久代是有可能发生GC的，不过条件很苛刻，各个虚拟机也没有很好的进行实现。





## 对象的实例化内存布局与访问定位

### 对象的实例化

#### 	创建对象的方式

1. new 关键字
2. class.newInsatnce			
3. Contructor的newInstance
4. 使用clone
5. 使用反序列化
6. 第三方库：Objenesjs

#### 创建对象的步骤

1. 判断对象的类是否有加载、连接、初始化
2. 为对象开辟内存空间
3. 处理并发安全问题
4. 初始化分配的空间--所有属性设置默认值，保证对象不赋值时可以直接使用
5. 设置对象的对象头
6. 显示初始化（代码块中初始化、构造器中初始化）

![image-20210124221734023](D:\workspace\note\image\image-20210124221734023.png)

#### 对象的内存布局

1. **对象头**

   - 运行时元数据MarkWord
     1. 哈希值 hascode
     2. GC分代年龄
     3. 锁状态标志
     4. 线程持有的锁
     5. 偏向线程ID
     6. 偏向时间戳
   - 类型指针：  指向类元数据InstanceClass，确定该对象的类型
   - 如果是数组，则还需要记录数组的长度

2. **示例数据（Instance Data）**

   - 他是对象真正存储的有效信息，包括程序代码中定义的各种类型的字段（包括从父类继承的及本身的字段）

   - 规则
     1. 相同宽度的字段总数被分配在一起
     2. 父类中定义的变量会出现在自身变量之前
     3. 如果CompactField参数为true，子类的窄变量可能会插入到父类变量的空隙

3. **对齐填充（Padding）**

   不是必须的，也没特别含义，仅仅起到占位符作用

![image-20210221111653506](D:\workspace\note\image\image-20210221111653506.png)





## 执行引擎

**将字节码指令解释/编译为对应平台上的本地机器指令** 也称作为**后端编译**

​	**解释器**

​	当JAVA虚拟机启动时，会根据预定义的规范，对字节码采用逐行解书的方式执行，将每条字节码内容翻译为对应平台的本地机器指令

​	**JIT（JSUT IN TIME）编译器**

​	就是虚拟机将源代码直接翻译成本地机器平台相关的机器语言

​	**字节码**









# GC

### 如何定位垃圾

1. 引用计数法

2. 可达性分析算法（Reachability Analysis）

   GC roots:**线程变量、静态变量、常量池、JNI引用对象(即Native方法)、class对象**

​	

### 常见的垃圾收集算法

- 分代收集理论（Generational Collection）

- Mark-Sweep (标记清除)

  对需要回收的对象进行标记，然后再进行清除

  缺点：

  1. 执行效率不稳定，因为大量对象都是进行回收的，所以需要进行大量标记和清除动作，导致标记和清除动作效率随着对象增多而降低
  2. 内存碎片化，空间不连续

- Semispace Coping(拷贝)

  将活着的对象复制到另外一个空间上面去，从而回收整个下半区。实现简单，运行效率也比较高

  缺点：

  1. 比较浪费空间，老年代不能采用这个算法

- Mark-Compact(标记整理) 

  对所有活着的对象进行标记，然后向内存的另外一端移动，之后对边界外的空间进行整理

  缺点：

  1. 移动存活的对象必须要STW(Stop The Word),停顿时间更长。

### JVM分代模型

1. 部分垃圾回收器使用模型
2. 新生代、老年代、永久代(1.7)/元数据区(1.8) Metaspace
   1. 永久代数据  --Class
   2. 永久代必须制定大小限制、元数据区可以设置也可以不设置（受限于物理内存）
   3. 字符串常量1.7-永久代、1.8元数据
   4. MeathArea逻辑概念，永久区、元数据区
3. 新生代 =Eden+2个suvivor区 Minor GC
   1. YGC之后，大多数对象会被回收，活着的对象进入suvivor0
   2. 再次YGC、活着的对象eden+s0->s1
   3. 再次YGC、活着对对象eden+s1->s0
   4. 年龄足够->老年代
4. 老年代
   1. 顽固分子
   2. 老年代满了  Major GC
5. GC Tuning(Generation)
   1. 尽量减少FullGc

**其中：跨代引用是有新生代的记忆集（Remembered Set）来进行标识，Minor GC的时候，这部分也会被加入的GCRoots里面**



### 常见的垃圾回收器

1. serial年轻代串行回收，只有一个线程，性能比较低

2. PS（Parallel Scavenge）并行回收---**多线程**，与ParNew 区别是：主要关注吞吐量,即：**减少运行垃圾收集的时间**

3. ParNew 年轻代配合CMS的并行回收

4. SerialOld  老年代串行回收，与serial搭配使用

5. ParallelOld PS的老年代收集器。

6. CMS(Concurrent Mark Sweep) 老年代，并发的。垃圾回收和应用程序同时运行，降低STW的时间。其一般跟**ParNew**一起搭配使用，当其预留空间放不下用户进程分配对象的需求的话，则不得不冻结所有用户进程，启用**Serial Old**收集器进行老年代的垃圾回收

   步骤包括：

   1. **初始标记**（CMS initial mark）

      仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿（STW -Stop the world）

   2. **并发标记（**CMS concurrent mark）

      从 GC Root 开始对堆中对象进行可达性分析，找到存活对象，它在整个回收过程中耗时最长，不需要停顿

   3. **重新标记**（CMS remark）

      为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿(STW)。这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短

   4. **并发清除**（CMS concurrent sweep）不需要停顿

   ![](D:\workspace\note\image\CMS垃圾回收过程.jpg)

   **CMS三个明显的缺点：**

   （2）CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。在JDK1.5的默认设置下，CMS收集器当老年代使用了68%的空间后就会被激活；

   （2）CMS收集器无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。在JDK1.5的默认设置下，CMS收集器当老年代使用了68%的空间后就会被激活。

   （3）CMS是基于“标记-清除”算法实现的收集器，手机结束时会有大量空间碎片产生。空间碎片过多，可能会出现老年代还有很大空间剩余，但是无法找到足够大的连续空间来分配当前对象，不得不提前出发FullGC。

   CMS**相关参数**

   ```
   #使用CMS垃圾收集器
   -XX:UseConcMarkSweepGC              
   
   #每一次FullGC之后进行一次碎片整理（默认开启） Java9废弃参数
   -XX:+UseCMSCompactAtFullCollection 
   
   #要求CMS执行若干次后不执行碎片整理，在进入FullGC前进行碎片整理，
   #（默认是0，表示每次进入FullGC前都进行垃圾整理
   -XX:+CMSFullGCsBeforeCompaction  
   
   #配置老年代使用率触发CMS，JDK-5是62% JDK-6是92%
   -XX:CMSInitiatingOccu-pancyFraction    
   
   #是否启用类卸载功能，默认是不允许
   -XX:+CMSClassUnloadingEnabled 
   
   #是否清理永久代，在java6过后就废弃了这个参数，如果添加启动项目会提示
   #Please use CMSClassUnloadingEnabled in place of CMSPermGenSweepingEnabled in the future
   
   -XX:+CMSPermGenSweepingEnabled
   ```

   

7. **G1(10ms) 逻辑分代，物理不分代**

   ## **1. 分区（Region）**

   G1采取了不同的策略来解决并行、串行和CMS收集器的碎片、暂停时间不可控制等问题——G1将整个堆分成相同大小的**分区（Region）**，如下图所示。

   ![](D:\workspace\note\image\G1垃圾回收.jpg)

   可参考

   [可能是最全面的G1学习笔记]: https://zhuanlan.zhihu.com/p/54048685

   

8. ZGC(1ms)

9. Shenandoah

   1.8默认的回收器是:PS+ParallelOld



### 了解生产环境的垃圾回收器组合

​	常用的JVM参数

​	-XX:+UseConcMarkSweepGC 使用CMS收集器（ParNew + CMS + Serial Old）

​	-XX:CMSFullGCsBeforeCompactiom=4 使用CMS在进行4次垃圾收集之后，进行一次内存碎片整理

​	-XX:+PrintCommandLineFlags 

​	-XX:+PrintFlagsFinal 最终参数值

​	-XX:+PrintFlagsInitial 默认参数值



![image-20210127200320244](D:\workspace\note\image\image-20210127200320244.png)