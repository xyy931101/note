# NIO

​	可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子

![](\image\reactor\NIO工作模型.png)

当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

所以，nonblocking IO的特点是用户进程需要**不断的主动询问**kernel数据好了没有。涉及到**多次的用户态内核态的切换**。

# select

应用进程想要通过select 去监控多个连接（也就是fd）的话需要经向大概如下的流程：

1、在调用select之前告诉select 应用进程需要监控哪些fd可读、可写、异常事件，这些分别都存在一个fd_set数组中。

2、然后应用进程调用select的时候把3个fd_set传给内核（这里也就产生了一次fd_set在用户空间到内核空间的复制），内核收到fd_set后对fd_set进行遍历，然后一个个去扫描对应fd是否满足可读写事件。

![](\image\reactor\select工作模型.jpg)

3、如果发现了有对应的fd有读写事件后，内核会把fd_set里没有事件状态的fd句柄清除，然后把有事件的fd返回给应用进程（这里又会把fd_set从内核空间复制用户空间）。

4、最后应用进程收到了select返回的活跃事件类型的fd句柄后，再向对应的fd发起数据读取或者写入数据操作。

![](\image\reactor\select工作流程2.jpg)

总的来说：select 提供一种可以用一个进程监控多个网络连接的方式，但也还遗留了一些问题，这些问题也是后来select面对高并发环境的性能瓶颈。

1、每调用一次select 就需要3个事件类型的fd_set需从用户空间拷贝到内核空间去，返回时select也会把保留了活跃事件的fd_set返回(从内核拷贝到用户空间)。当fd_set数据大的时候，这个过程消耗是很大的。

2、select需要逐个遍历fd_set集合 ，然后去检查对应fd的可读写状态，如果fd_set 数据量多，那么遍历fd_set 就是一个比较耗时的过程。

3、fd_set是个集合类型的数据结构有长度限制,32位系统长度1024,62位系统长度2048，这个就限制了select最多能同时监控1024个连接。

# poll

吸取了select的教训，poll模式就不再使用数组的方式来保存自己所监控的fd信息了，poll模型里面通过使用链表的形式来保存自己监控的fd信息，正是这样poll模型里面是没有了连接限制，可以支持高并发的请求。

和select还有一点不同的是保存在链表里的需要监控的fd信息采用的是pollfd的文件格式，select 调用返回的fd_set是只包含了上次返回的活跃事件的fd_set集合，下一次调用select又需要把这几个fd_set清空，重新添加上自己感兴趣的fd和事件类型，而poll采用的pollfd 保存着对应fd需要监控的事件集合，也保存了一个当返回于激活事件的fd集合。 所以重新发请求时不需要重置感兴趣的事件类型参数。

# epoll

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



然后epoll并不是像select一样去遍历事件列表，然后逐个轮询的监控fd的事件状态，而是事先就建立了fd与之对应的回调函数，当事件激活后主动回调callback函数，这也就避免了遍历事件列表的这个操作,所以epoll并不会像select和poll一样随着监控的fd变多而效率降低，这种事件机制也是epoll要比select和poll高效的主要原因。

# Reactor 是什么

1. 事件驱动（event handling）
2. 可以处理一个或多个输入源（one or more inputs）
3. 通过Service Handler同步的将输入事件（Event）采用多路复用分发给相应的Request Handler（多个）处理

## 单Reactor单线程模型

![](D:\workspace\note\image\reactor\reactor单线程.png)

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

![](D:\workspace\note\image\reactor\单reactor多线程.png)

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

## 多Reactor多线程模型

![](\reactor\多Reactor多线程模型.png)

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

