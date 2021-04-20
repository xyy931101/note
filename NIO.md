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

![](D:\workspace\note\image\reactor\多Reactor多线程模型.png)

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

