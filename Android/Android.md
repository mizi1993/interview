#View工作原理
##ViewRoot和DecorView
ViewRoot对应ViewRootImpl类，是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建出来后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联。
绘制流程都是从`ViewRootImpl.performTraversals()`开始，依次调用performMeasure(),performLayout(),performDraw()。这些方法会调用DecorView的measure()方法，`measure()`又会调用`onMeasure()`完成子View的measure。其余的流程和Measure类似。
DecorView实际上是一个FrameLayout，View层的事件都要先经过DecorView然后才传递到View。

##MeasureSpec
MeasureSpec是一个32位int值，高2位代表SpecMode(测量模式)，低30位代表SpecSize(测量大小)。
SpecMode有三类
- UNSPECIFIED 父容器不对View做限制，要多大有多大
- EXACTLY 父容器已经测量出了精确大小，这时候View的最终大小就是SpecSize所指定的值，它对应于match_parent和精确值这两种
- AT\_MOST 父容器指定了一个SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。对应于wrap\_content

对DecorView来说，其MeasureSpec由窗口的尺寸和其自身的LayoutParams来共同决定；而普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来决定。

##View的工作流程
###measure
如果是View，测量自己就可以了。如果是一个ViewGroup，除了完成自己的测量外，还会遍历去调用所有子元素的measure方法，各个子元素再递归执行这个流程

###layout
layout的作用是ViewGroup用来确定子元素的位置，当ViewGroup的位置被确定之后，他在onLayout中会遍历所有的子元素并调用其layout方法，在layout方法中onLayout又会被调用。
layout方法确定View本身位置，onLayout方法确定所有子元素位置。

###draw过程
View的draw过程有如下几步
1. background.draw()
2. onDraw()
3. dispatchDraw()
4. onDrawScrollBars()
**有一个特殊的方法 ： setWillNotDraw，如果一个View不需要绘制任何内容，则可以将这个标记为设置为true。ViewGroup默认开启了这个标志位，所以ViewGroup的OnDraw方法不会进行绘制。**


#Handler
Handler机制相关的类有
- Handler
- Looper
- MessageQueue
- Message
而Looper是通过管道(pipe)实现的。

>关于管道，简单来说，管道就是一个文件。
在管道的两端，分别是两个打开文件文件描述符，这两个打开文件描述符都是对应同一个文件，其中一个是用来读的，别一个是用来写的。
一般的使用方式就是，一个线程通过读文件描述符中来读管道的内容，当管道没有内容时，这个线程就会进入等待状态，而另外一个线程通过写文件描述符来向管道中写入内容，写入内容的时候，如果另一端正有线程正在等待管道中的内容，那么这个线程就会被唤醒。这个等待和唤醒的操作是如何进行的呢，这就要借助Linux系统中的epoll机制了。 Linux系统中的epoll机制为处理大批量句柄而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著减少程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。

##初始化完成的工作
- Java层，创建Looper对象，Looper的构造函数汇创建消息队列MessageQueue对象。MessageQueue对象用于储存消息队列，管理消息。
- Native层，MessageQueue创建时，会调用JNI函数，初始化NativeMessageQueue。NativeMessageQueue会初始化Looper对象，Looper对象的作用就是，当Java层的消息队列没有消息时，就使Android线程进入等待状态，而当Java层的消息队列中来了消息之后，就唤醒Android应用程序的主线程来处理这个消息。

##发送消息
消息发送最终会调用到`MessageQueue.enqueueMessage()`，根据message.when来确定插入的位置并把消息加入到队列中，分为以下两种情况
- 消息队列为空：此时app主线程应处于等待状态，需要调用wake唤醒。
- 消息队列不为空：不需要唤醒主线程。

##程序为什么不会被Looper.loop()卡死
进程：每个app运行时首先创建一个进程，该进程是由Zygote fork出来的，用于Activity/Service的运行。
线程：线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片。

##主要调用关系
```
            +------------------+
            |      Handler     |
            +----^--------+----+
                 |        |
        dispatch |        | send
                 |        |
                 |        v
                +----+ <---+
                |          |
                |  Looper  |
                |          |
                |          |
                +---> +----+
                  ^      |
             next |      | enqueue
                  |      |
         +--------+------v----------+
         |       MessageQueue       |
         +--------+------+----------+
                  |      |
  nativePollOnce  |      |   nativeWake
                  |      |
+----------------------------------------------+ Native Layer
                  |      |
       pollOnce   |      |  wake
                  |      |
         +--------v------v--------+
         |   NativeMessageQueue   |
         +--------+------+--------+
                  |      |
         pollOnce |      |  wake
         pollInner|      |  awoken
                  |      |
              +---v------v---+
              |    Looper    |
              +-+----------+-+
                |          |
     epoll_wait |          |  wake
  +-------------v-+      +-v--------------+
  |mWakeReadPipeFd|      |mWakeWritePipeFd|
  +-------------^-+      +-+--------------+
                |          |
          read  |          | write
                |          |
              +-+----------v-+
              |     Pipe     |
              +--------------+
```

#Binder
##使用以及上层原理
直观来说，Binder是Android中的一个类，他实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/nomder。从Android Framework的角度来说，Binder是ServiceManager连接各种Manager和响应ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个对象，客户端就可以获取服务端提供的服务或数据，这里的服务包括普通服务和基于AIDL的服务。
Android开发中，Binder主要用于Service，包括AIDL和Messenger，其中普通Service中的Binder不涉及进程间通信，所以较为简单，无法触及Binder的核心，而Messenger的底层其实是AIDL。




`Toast.makeText().show()`是将Toast加入显示队列