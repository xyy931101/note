# 磁盘IO

![](image\reactor\磁盘ID工作模型.png)

具体步骤：

       当应用程序调用read接口时，操作系统检查内核缓冲区中是否存在需要的数据，如果存在，就直接从内核缓存中直接返回，否则从磁盘中读取，然后缓存至操作系统的缓存中。


       当应用程序调用write接口时，将数据直接从用户地址空间复制到内核地址空间的缓存中，这时对用户程序来说，写操作已经完成了，至于什么时候写入磁盘中，由操作系统决定，除非显示调用sync同步命令。

 

## 内核空间和用户空间

我们电脑上跑着的应用程序，其实是需要经过操作系统，才能做一些特殊操作，如磁盘文件读写、内存的读写等等。因为这些都是比较危险的操作，不可以由应用程序乱来，只能交给底层操作系统来。

因此，操作系统为每个进程都分配了内存空间，一部分是用户空间，一部分是内核空间。内核空间是操作系统内核访问的区域，是受保护的内存空间，而用户空间是用户应用程序访问的内存区域。 以32位操作系统为例，它会为每一个进程都分配了4G(2的32次方)的内存空间。

内核空间：主要提供进程调度、内存分配、连接硬件资源等功能
用户空间：提供给各个程序进程的空间，它不具有访问内核空间资源的权限，如果应用程序需要使用到内核空间的资源，则需要通过系统调用来完成。进程从用户空间切换到内核空间，完成相关操作后，再从内核空间切换回用户空间。

## 用户态和内核态

如果进程运行于内核空间，被称为进程的内核态。
如果进程运行于用户空间，被称为进程的用户态。

## 什么是上下文切换

什么是上下文

> CPU 寄存器，是CPU内置的容量小、但速度极快的内存。而程序计数器，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此叫做CPU上下文。

什么是上下文切换

> 它是指，先把前一个任务的CPU上下文（也就是CPU寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。

一般我们说的上下文切换，就是指内核（操作系统的核心）在CPU上对进程或者线程进行切换。进程从用户态到内核态的转变，需要通过系统调用来完成。系统调用的过程，会发生CPU上下文的切换。

![](image\io\上下文切换.png)

## 虚拟内存

现代操作系统使用虚拟内存，即虚拟地址取代物理地址，使用虚拟内存可以有2个好处：

1.虚拟内存空间可以远远大于物理内存空间
2.多个虚拟内存可以指向同一个物理地址
正是多个虚拟内存可以指向同一个物理地址，可以把内核空间和用户空间的虚拟地址映射到同一个物理地址，这样的话，就可以减少IO的数据拷贝次数，示意图如下
![](image\io\系统虚拟内存.png)

## DMA技术

DMA，英文全称是Direct Memory Access，即直接内存访问。DMA本质上是一块主板上独立的芯片，允许外设设备和内存存储器之间直接进行IO数据传输，其过程不需要CPU的参与。

简单的说它就是帮住CPU转发一下IO请求以及拷贝数据，那为什么需要它呢？其实主要是效率问题。它帮忙CPU做事情，这时候，CPU就可以闲下来去做别的事情，提高了CPU的利用效率。大白话解释就是，CPU老哥太忙太累啦，所以他找了个小弟（名叫DMA） ，替他完成一部分的拷贝工作，这样CPU老哥就能着手去做其他事情。

下面看下DMA具体是做了哪些工作

![](image\io\DMA工作流程.png)

1.用户应用程序调read函数，向操作系统发起IO调用，进入阻塞状态等待数据返回
2.CPU接到指令后，对DMA控制器发起指令调度
3.DMA收到请求后，将请求发送给磁盘
4.磁盘将数据放入磁盘控制缓冲区并通知DMA
5.DMA将数据从磁盘控制器缓冲区拷贝到内核缓冲区
6.DMA向CPU发送数据读完的信号，CPU负责将数据从内核缓冲区拷贝到用户缓冲区
7.用户应用进程由内核态切回用户态，解除阻塞状态

# 如何实现零拷贝

零拷贝并不是没有拷贝数据，而是减少用户态、内核态的切换次数以及CPU拷贝次数；实现零拷贝主要有三种方式分别是
1.mmap + write
2.sendfile
3.带有DMA收集拷贝功能的sendfile

## mmap

mmap的函数原型如下

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

addr:指定映射的虚拟内存地址
length:映射的长度
prot:映射内存的保护模式
flags:指定映射的类型
fd:进行映射的文件句柄
offset:文件偏移量

mmap+write实现的零拷贝流程如下：

![](image\io\mmap+write工作流程.png)

1.用户进程通过调用mmap方法向操作系统内核发起IO调用，上下文从用户态切换至内核态
2.CPU利用DMA控制器，将数据从硬盘拷贝到内核缓冲区
3.上下文从内核态切换回用户态，mmap方法返回
4.用户进程通过调用write方法向操作系统内核再次发起IO调用，上下文从用户态切换至内核态
5.CPU将内核缓冲区的数据拷贝到socket缓冲区
6.CPU利用DMA控制器，将数据从socket缓冲器拷贝到网卡，上下文从内核态切换至用户态，write方法返回

## sendfile

sendfile是Linux2.1版本后内核引入 的一个系统调用函数，原型如下

```cpp
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

out_fd:为待写入内容的文件描述符
in_fd:为待读出内容的文件描述符
offset:文件偏移量
count:指定在fdout和fdin之间传输的字节数

sendfile表示在两个文件描述符之间传输数据，它是在操作系统内核中操作的，避免了数据从内核缓冲区和用户缓冲区之间的拷贝操作，因此可以使用它来实现零拷贝。sendfile实现的零拷贝流程如下：

![](image\io\sendfile工作流程.png)

1.用户进程发起sendfile系统调用，上下文从用户态切换至内核态
2.DMA控制器将数据从硬盘拷贝到内核缓冲区
3.CPU将读缓冲区中的数据拷贝到socket缓冲区
4.DMA控制器异步把数据从socket缓冲器拷贝到网卡
5.上下文从内核态切换至用户态，sendfile函数返回

## sendfile +DMA scatter/gather实现的零拷贝

linux2.4版本后，对sendfile做了优化升级，引入SG-DMA技术，其实就是对DMA拷贝加入了scatter/gather操作，它可以直接从内核空间缓冲区中将数据读取到网卡，这样的话还可以省去CPU拷贝。

![](image\io\零拷贝工作流程.png)

1.用户进程发起sendfile系统调用，上下文从用户态切换至内核态
2.DMA控制器将数据从磁盘拷贝到内核缓冲器
3.CPU把内核缓冲区中的文件描述符信息（包括内核缓冲区的内存地址和偏移量）直接发送到socket缓冲区
4.DMA控制器根据文件描述符信息直接把数据从内核缓冲区拷贝到网卡
5.上下文切换至用户态，sendfile返回

**可以发现sendfile + DMA scatter/gather实现的零拷贝发生了2次上下文切换以及2次数据拷贝，这就是真正的零拷贝技术，全程没有通过CPU来搬运数据，所有的数据都是通过DMA进行传输的。**

## 总结

- 传统 IO 执行的话需要 4 次上下文切换（用户态 -> 内核态 -> 用户态 -> 内核态 -> 用户态）和 4 次拷贝（磁盘文件 DMA 拷贝到内核缓冲区，内核缓冲区 CPU 拷贝到用户缓冲区，用户缓冲区 CPU 拷贝到 Socket 缓冲区，Socket 缓冲区 DMA 拷贝到协议引擎）。
- mmap 将磁盘文件映射到内存，支持读和写，对内存的操作会反映在磁盘文件上，**适合小数据量读写**，需要 4 次上下文切换（用户态 -> 内核态 -> 用户态 -> 内核态 -> 用户态）和3 次拷贝（磁盘文件DMA拷贝到内核缓冲区，内核缓冲区 CPU 拷贝到 Socket 缓冲区，Socket 缓冲区 DMA 拷贝到协议引擎）。
- sendfile 是将读到内核空间的数据，转到 socket buffer，进行网络发送，**适合大文件传输**，只需要 2 次上下文切换（用户态 -> 内核态 -> 用户态）和 2 次拷贝（磁盘文件 DMA 拷贝到内核缓冲区，内核缓冲区 DMA 拷贝到协议引擎）。
  

# java提供的零拷贝方式

## mmap

Java NIO有一个MappedByteBuffer的类可以用来实现内存映射。它的底层是调用的linux内核的mmap的API

```java
public class MmapTest {	
	public static void main(String[] args) {
    	try {
      		FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
       	 	MappedByteBuffer data = readChannel.map(FileChannel.MapMode.READ_ONLY, 0, 1024 * 1024 * 40);
       	 	FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        	//数据传输
       	 	writeChannel.write(data);
        	readChannel.close();
        	writeChannel.close();
    	}catch (Exception e){
        	System.out.println(e.getMessage());
    	}
	}
}
```
## sendfile

FileChannel的transferTo()/transferFrom()，底层就是sendfile() 系统调用函数。Kafka 这个开源项目就用到它，平时面试的时候，回答面试官为什么这么快，就可以提到零拷贝sendfile这个点

```java
public class SendFileTest {
    public static void main(String[] args) {
        try {
            FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
            long len = readChannel.size();
            long position = readChannel.position();
			FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        	//数据传输
        	readChannel.transferTo(position, len, writeChannel);
        	readChannel.close();
        	writeChannel.close();
    	} catch (Exception e) {
        	System.out.println(e.getMessage());
    	}
    }
}
```


# 网络IO

​	可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子

![](image\reactor\NIO工作模型.webp)

当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

所以，nonblocking IO的特点是用户进程需要**不断的主动询问**kernel数据好了没有。涉及到**多次的用户态内核态的切换**。

## BIO

同步并阻塞（传统阻塞型），服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销

![](image\io\BIO流程.png)

## NIO

同步非阻塞，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器[轮询](https://so.csdn.net/so/search?q=轮询&spm=1001.2101.3001.7020)到连接有I/O请求就进行处理

![](image\io\NIO原理.png)

### NIO概述

- NIO（Non-Blocking IO）是从Java 1.4版本开始引入的一个新的IO API，可以替代标准的Java IO API。NIO与原来的IO有同样的作用和目的，但是使用的方式完全不同，NIO支持面向缓冲区的、基于通道的IO操作。NIO将以更加高效的方式进行文件的读写操作。NIO可以理解为非阻塞IO,传统的IO的read和write只能阻塞执行，线程在读写IO期间不能干其他事情，比如调用socket.read()时，如果服务器一直没有数据传输过来，线程就一直阻塞，而NIO中可以配置socket为非阻塞模式
- NIO相关类都被放在java.nio包及子包下，并且对原java.io包中的很多类进行改写
- NIO有三大核心部分：Channel（通道）、Buffer（缓冲区）、Selector（选择器）
- Java NIO的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情
- 通俗理解：NIO是可以做到用一个线程来处理多个操作的。假设有1000个请求过来，根据实际情况，可以分配20或者80个线程来处理。不像之前的阻塞IO那样，非得分配1000个

### NIO和BIO的比较

- BIO以流的方式处理数据，而NIO以块的方式处理数据，块I/O的效率比流I/O高很多

- BIO是阻塞的，NIO则是非阻塞的

- BIO基于字节流和字符流进行操作，而NIO基于Channel（通道）和Buffer（缓冲区）进行操作，数据总是从通道

  读取到缓冲区中，或者从缓冲区写入到通道中。Selector（选择器）用于监听多个通道的事件（比如：连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道

### NIO三大核心原理示意图

NIO有三大核心部分：Channel（通道）、Buffer（缓冲区）、Selector（选择器）

#### Buffer缓冲区：

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。相比较直接对数组的操作，Buffer API更加容易操作和管理

#### Channel（通道）：

Java NIO的通道类似流，但又有些不同：既可以从通道中读取数据，又可以写数据到通道。但流的（input或output）读写通常是单向的。通道可以非阻塞读取和写入通道，通道可以支持读取或写入缓冲区，也支持异步地读写

#### Selector选择器：

Selector是一个Java NIO组件，可以能够检查一个或多个NIO通道，并确定哪些通道已经准备好进行读取或写入。这样，一个单独的线程可以管理多个channel，从而管理多个网络连接，提高效率


## AIO

异步非阻塞，服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由操作系统先完成了再通知服务器应用去启动线程进行处理，一般适用于连接数较多且连接时间较长的应用

## 多路复用器

### select

应用进程想要通过select 去监控多个连接（也就是fd）的话需要经向大概如下的流程：

1、在调用select之前告诉select 应用进程需要监控哪些fd可读、可写、异常事件，这些分别都存在一个fd_set数组中。

2、然后应用进程调用select的时候把3个fd_set传给内核（这里也就产生了一次fd_set在用户空间到内核空间的复制），内核收到fd_set后对fd_set进行遍历，然后一个个去扫描对应fd是否满足可读写事件。

![](image\reactor\select工作模型.png)

3、如果发现了有对应的fd有读写事件后，内核会把fd_set里没有事件状态的fd句柄清除，然后把有事件的fd返回给应用进程（这里又会把fd_set从内核空间复制用户空间）。

4、最后应用进程收到了select返回的活跃事件类型的fd句柄后，再向对应的fd发起数据读取或者写入数据操作。

总的来说：select 提供一种可以用一个进程监控多个网络连接的方式，但也还遗留了一些问题，这些问题也是后来select面对高并发环境的性能瓶颈。

1、每调用一次select 就需要3个事件类型的fd_set需从用户空间拷贝到内核空间去，返回时select也会把保留了活跃事件的fd_set返回(从内核拷贝到用户空间)。当fd_set数据大的时候，这个过程消耗是很大的。

2、select需要逐个遍历fd_set集合 ，然后去检查对应fd的可读写状态，如果fd_set 数据量多，那么遍历fd_set 就是一个比较耗时的过程。

3、fd_set是个集合类型的数据结构有长度限制,32位系统长度1024,62位系统长度2048，这个就限制了select最多能同时监控1024个连接。

### poll

吸取了select的教训，

1. poll模型里面通过使用**链表的形式**来保存自己监控的fd信息，正是这样poll模型里面是没有了连接限制，可以支持高并发的请求。
2. poll采用的pollfd 保存着对应fd需要监控的事件集合，也保存了一个当返回于激活事件的fd集合。 所以重新发请求时不需要重置感兴趣的事件类型参数。

### epoll

不同于select 和poll的直接调用方式，epoll采用的是一组方法调用的方式，它的工作流程大致如下：

**1、创建内核事件表（epoll_create）**。

这里主要是向内核申请创建一个fd的文件描述符作为内核事件表（B+树结构的文件，没有数量限制），这个描述符用来保存应用进程需要监控哪些fd和对应类型的事件。

![img](https://pic3.zhimg.com/80/v2-ac259283c26446651ff4874c31fd4fbe_1440w.jpg)





**2、添加或移出监控的fd和事件类型（epoll_ctl）。**

调用此方法可以是向内核的内核事件表 动态的添加和移出fd 和对应事件类型。

![img](https://pic4.zhimg.com/80/v2-f6d66bd470807631267e6015f089b2b3_1440w.jpg)





**3、epoll_wait 绑定回调事件**

内核向事件表的fd绑定一个回调函数。

![img](https://pic3.zhimg.com/80/v2-a8d965d0be91d79753bb703700532376_1440w.jpg)





当监控的fd活跃时，会调用callback函数把事件加到一个活跃事件队列里;

![img](https://pic1.zhimg.com/80/v2-d3de542d960187c9e135b2f9b58ac4f0_1440w.jpg)





最后在epoll_wait 返回的时候内核会把活跃事件队列里的fd和事件类型返回给应用进程。

![img](https://pic1.zhimg.com/80/v2-cedb63e8810026afe11a1a361dc98a64_1440w.jpg)

最后，从epoll整体思路上来看，采用事先就在内核创建一个事件监听表，后面只需要往里面添加移出对应事件，因为本身事件表就在内核空间，所以就避免了向select、poll一样每次都要把自己需要监听的事件列表传输过去，然后又传回来，这也就避免了事件信息需要在用户空间和内核空间相互拷贝的问题。

![](image\reactor\epoll工作模型.png)

然后epoll并不是像select一样去遍历事件列表，然后逐个轮询的监控fd的事件状态，而是事先就建立了fd与之对应的回调函数，当事件激活后主动回调callback函数，这也就避免了遍历事件列表的这个操作,所以epoll并不会像select和poll一样随着监控的fd变多而效率降低，这种事件机制也是epoll要比select和poll高效的主要原因。

# Reactor 

1. 事件驱动（event handling）
2. 可以处理一个或多个输入源（one or more inputs）
3. 通过Service Handler同步的将输入事件（Event）采用多路复用分发给相应的Request Handler（多个）处理

## 单Reactor单线程模型

> 单线程：建立连接（Acceptor）、监听accept、read、write事件（Reactor）、处理事件（Handler）都只用一个单线程。

![](image\reactor\reactor单线程.png)

```java

  /**
    * 等待事件到来，分发事件处理
    */
  class Reactor implements Runnable {
      private Reactor() throws Exception {

          SelectionKey sk =
                  serverSocket.register(selector,
                          SelectionKey.OP_ACCEPT);
          // attach Acceptor 处理新连接
          sk.attach(new Acceptor());
      }

      public void run() {
          try {
              while (!Thread.interrupted()) {
                  selector.select();
                  Set selected = selector.selectedKeys();
                  Iterator it = selected.iterator();
                  while (it.hasNext()) {
                      it.remove();
                      //分发事件处理
                      dispatch((SelectionKey) (it.next()));
                  }
              }
          } catch (IOException ex) {
              //do something
          }
      }

      void dispatch(SelectionKey k) {
          // 若是连接事件获取是acceptor
          // 若是IO读写事件获取是handler
          Runnable runnable = (Runnable) (k.attachment());
          if (runnable != null) {
              runnable.run();
          }
      }
  }

```

```java

  /**
    * 连接事件就绪,处理连接事件
    */
  class Acceptor implements Runnable {
      @Override
      public void run() {
          try {
              SocketChannel c = serverSocket.accept();
              if (c != null) {// 注册读写
                  new Handler(c, selector);
              }
          } catch (Exception e) {

          }
      }

```

这是最基本的单Reactor单线程模型。其中Reactor线程，负责多路分离套接字，有新连接到来触发connect 事件之后，交由Acceptor进行处理，有IO读写事件之后交给hanlder 处理。

Acceptor主要任务就是构建handler ，在获取到和client相关的SocketChannel之后 ，绑定到相应的hanlder上，对应的SocketChannel有读写事件之后，基于racotor 分发,hanlder就可以处理了（所有的IO事件都绑定到selector上，有Reactor分发）。

该模型 适用于处理器链中业务处理组件能快速完成的场景。不过，这种单线程模型不能充分利用多核资源，所以实际使用的不多



## 单Reactor多线程模型

> 一个线程 + 一个线程池： 单线程：建立连接（Acceptor）和 监听accept、read、write事件（Reactor），复用一个线程。 工作线程池：处理事件（Handler），由一个工作线程池来执行业务逻辑，包括数据就绪后，用户态的数据读写。



![](image\reactor\单reactor多线程.png)

相对于第一种单线程的模式来说，在处理业务逻辑，也就是获取到IO的读写事件之后，交由线程池来处理，这样可以减小主reactor的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。

```java
/**
    * 多线程处理读写业务逻辑
    */
class MultiThreadHandler implements Runnable {
    public static final int READING = 0, WRITING = 1;
    int state;
    final SocketChannel socket;
    final SelectionKey sk;

    //多线程处理业务逻辑
    ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public MultiThreadHandler(SocketChannel socket, Selector sl) throws Exception {
        this.state = READING;
        this.socket = socket;
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
        socket.configureBlocking(false);
    }

    @Override
    public void run() {
        if (state == READING) {
            read();
        } else if (state == WRITING) {
            write();
        }
    }

    private void read() {
        //任务异步处理
        executorService.submit(() -> process());

        //下一步处理写事件
        sk.interestOps(SelectionKey.OP_WRITE);
        this.state = WRITING;
    }

    private void write() {
        //任务异步处理
        executorService.submit(() -> process());

        //下一步处理读事件
        sk.interestOps(SelectionKey.OP_READ);
        this.state = READING;
    }

    /**
        * task 业务处理
        */
    public void process() {
        //do IO ,task,queue something
    }
}

```

## 主从Reactor模式

> 三个线程池： 主线程池：建立连接（Acceptor），并且将accept事件注册到从线程池。 从线程池：监听accept、read、write事件（Reactor），包括等待数据就绪时，内核态的数据I读写。 工作线程池：处理事件（Handler），由一个工作线程池来执行业务逻辑，包括数据就绪后，用户态的数据读写

![](image\reactor\多Reactor多线程模型.png)

第三种模型比起第二种模型，是将Reactor分成两部分，

1. mainReactor负责监听server socket，用来处理新连接的建立，将建立的socketChannel指定注册给subReactor。
2. subReactor维护自己的selector, 基于mainReactor 注册的socketChannel多路分离IO读写事件，读写网 络数据，对业务处理的功能，另其扔给worker线程池来完成。

```java
/**
    * 多work 连接事件Acceptor,处理连接事件
    */
  class MultiWorkThreadAcceptor implements Runnable {
      // cpu线程数相同多work线程
      int workCount =Runtime.getRuntime().availableProcessors();
      SubReactor[] workThreadHandlers = new SubReactor[workCount];
      volatile int nextHandler = 0;
      public MultiWorkThreadAcceptor() {
          this.init();
      }
      public void init() {
          nextHandler = 0;
          for (int i = 0; i < workThreadHandlers.length; i++) {
              try {
                  workThreadHandlers[i] = new SubReactor();
              } catch (Exception e) {
              }
          }
      }

      @Override
      public void run() {
          try {
              SocketChannel c = serverSocket.accept();
              if (c != null) {// 注册读写
                  synchronized (c) {
                      // 顺序获取SubReactor，然后注册channel 
                      SubReactor work = workThreadHandlers[nextHandler];
                      work.registerChannel(c);
                      nextHandler++;
                      if (nextHandler >= workThreadHandlers.length) {
                          nextHandler = 0;
                      }
                  }
              }
          } catch (Exception e) {
          }
      }
}
```

```java
 /**
    * 多work线程处理读写业务逻辑
    */
  class SubReactor implements Runnable {
      final Selector mySelector;

      //多线程处理业务逻辑
      int workCount =Runtime.getRuntime().availableProcessors();
      ExecutorService executorService = Executors.newFixedThreadPool(workCount);

      public SubReactor() throws Exception {
          // 每个SubReactor 一个selector 
          this.mySelector = SelectorProvider.provider().openSelector();
      }

      /**
        * 注册chanel
        *
        * @param sc
        * @throws Exception
        */
      public void registerChannel(SocketChannel sc) throws Exception {
          sc.register(mySelector, SelectionKey.OP_READ | SelectionKey.OP_CONNECT);
      }

      @Override
      public void run() {
          while (true) {
              try {
              //每个SubReactor 自己做事件分派处理读写事件
                  selector.select();
                  Set<SelectionKey> keys = selector.selectedKeys();
                  Iterator<SelectionKey> iterator = keys.iterator();
                  while (iterator.hasNext()) {
                      SelectionKey key = iterator.next();
                      iterator.remove();
                      if (key.isReadable()) {
                          read();
                      } else if (key.isWritable()) {
                          write();
                      }
                  }
              } catch (Exception e) {

              }
          }
      }

      private void read() {
          //任务异步处理
          executorService.submit(() -> process());
      }

      private void write() {
          //任务异步处理
          executorService.submit(() -> process());
      }

      /**
        * task 业务处理
        */
      public void process() {
          //do IO ,task,queue something
      }
}
```

