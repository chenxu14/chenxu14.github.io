---
layout: default
title:  "Netty池化管理"
---

# Netty池化管理
Netty的池化管理主要围绕这样几个方面，包括对象池、内存池以及线程池。

### 一、对象池
想要实现一个java的对象池其实并不困难，比如最简单的方法我们可以借助于java的BlockingQueue来实现最基本的对象出队入队逻辑，或者随便定义一个集合通过读写锁的方式来对资源访问进行控制。然而不好的一点是，当并发线程数比较大的时候会带来很大程度的锁同步开销，产生一些overhead，netty在实现对象池的时候充分考虑了这一因素，可以尽量规避锁同步带来的资源消耗。

![Obj Pool](/images/netty_recyler.png)

首先在每个线程内部开启一个ThreadLocal，ThreadLocal内部封装了一个elements数组，用来存储待缓存的对象实例。由于这部分数据只是线程内部持有，因为访问和写入不会带来任何锁同步方面的开销，然而会有这样一个问题，当前线程所创建的缓存对象有可能被其他线程所使用，当其他线程使用完成之后该如何对该对象进行回收处理?我们不能把对象在直接放回elements数组里，由于是不同的线程，这样处理需要加锁同步。netty的做法是针对每一个使用缓存对象的线程开启一个弱引用队列，当该线程使用完对象之后，直接将其回收到它自己的队列里，这样从其他线程的视角来看，对象的回收过程同样不涉及锁抢占逻辑，因为每个线程都对应自己的回收队列。

采用弱引用的好处是，当前线程可以感知到其他使用缓存对象的线程是否已经运行结束退出，通过判断对象的引用值是否为null就可以。如果为空说明目标线程已经运行结束并被gc处理，这个时候当前线程需要做的一件事情就是把目标线程回收队列中的数据全部转存到自己的elements数组里由于这个过程同样是在单线程内部来完成的，因此同样不涉及锁抢占方面的开销。

还有一个应用场景就是当前线程的elements数组已无可用元素，它所存储的缓存对象已全被使用，并且其他线程也没有运行结束退出，这个时候需当前线程要把其他线程中可回收的对象实例转存到自己的elements数组里。在每一个回收队列的内部，数据是按照分段的方式进行组织的，每一个数据段封装成一个Link，其内部包含了一个对象数组，以及一个readIndex指针，该指针主要用来标识在该位置之前的数据记录已经全部转移到了elements数组里，同时Link对象在类结构上继承了java的AtomicInteger，这样便可以通过它的value值来确定当前Link的容量。

另外之所以采用分段的方式进行管理主要是为了便于对象做垃圾回收，当Link里面的对象实例全部转存到elements数组之后，它所占据的内存空间便可以通过GC释放掉，从整个转存的过程来看依然是不涉及锁抢占逻辑的，因为readIndex索引只有当前线程会去更新。

### 二、内存池
netty的内存池管理很大程度上参考了facebook的jemalloc实现，在执行内存分配的时候主要满足了这样几个原则：
  1. 多线程情况下尽量降低内存分配的锁抢占粒度。
  2. 另外需要保证空间分配的连续性，尽量不产生内存碎片。
接下来我们看一下，针对每一个方面，netty是如何做到的。

##### PoolThreadCache
首先，同对象池管理相类似，优先使用线程的本地cache。

![PoolThreadCache](/images/PoolThreadCache.png)

线程本地cache在netty里面主要是通过PoolThreadCache这个类来封装的，可以看到这个类里面封装了很多个缓存队列，每一个队列用来存储固定大小的entry，同时根据entry大小的不同又将队列划分了成多个分组。
  - small分组有4个队列用来存储512字节到4KB之间的数据
  - tiny分组有32个队列，用来存储16字节到496字节之间的数据
  - 最后normal分组有3个队列用来存储8kb到32kb之间的数据
这样每次执行内存分配的时候，只需要找到与目标空间相近的一个分组，然后从分组队列中获取相应的entry即可。

##### PoolArena
如果本地cache已经没有可用空间，需要基于PoolArena来进行内存分配管理。

![PoolArena](/images/PoolArena.png)

在初始化内存池的时候，Netty会为我们构建多个PoolArena实例，然后基于轮训的机制将这些PoolArena依次分配给每一个使用线程，由于资源申请的加锁粒度是PoolArena层级的，这样如果不同的线程采用不同的PoolArena进行申请便不会存在资源抢占冲突。相当于将一把锁在水平层面拆分成多个，跟NN的按目录拆锁相类似。

##### PoolChunk
然后看一下在内存碎片方面netty是怎样做到规避的。首先看一下netty的内存管理结构。

![PoolChunk](/images/PoolChunk.png)

在物理空间上，netty是采用划分chunk的方式来进行管理的，每一个Chunk对应的存储空间为16M，Chunk的内部是按照分页的方式进行内存组织的，每个分页默认大小为8Kb，这样16M的空间一共可以划分2048个page。
如果要申请的内存空间大小是大于一个分页的，netty主要基于伙伴分配算法来进行分配管理。算法的大致实现逻辑是这样的：首先将Chunk的空间地址映射成一个平衡二叉树，每个叶子节点对应一个page，这样2048个page所映射成的二叉树一共有11个层级，每个层级的节点对应的存储容量为8kb * 2的11-n次方。在执行分配过程中，主要基于深度优先遍历，找到符合目标容量需求的二叉树节点，然后基于该节点的所有叶子节点来构建ByteBuffer，从而确保分配空间是连续的，执行完分配操作以后，需要对目标节点及其所有祖先节点进行回溯，来更新每个节点的容量信息。

##### PoolSubpage
另外一种情况，如果申请的内存空间低于一个page，那么netty会将page做分组拆分管理。

![PoolSubpage](/images/PoolSubpage.png)

将每个page划分成更细粒度的subPage，每个subpage用来存储固定大小的空间数据，同时在subpage的内部还会封装一个bitmap，用来标识每块空间的分配情况，比如1表示这块空间已经分配出去了，0表示还尚未分配，另外相同分组的subpage是通过双向链表进行关联的，便于遍历引用方便。

### 三、线程池
最后看一下netty的线程池管理。

![ThreadPool](/images/threadpool.png)

关于线程池管理这块目前调研的还不是很深，目前只知道netty的线程池模型跟JDK原生自带的线程池相比是有一些变动的，首先看一下JDK原生的线程池，线程池的内部主要是封装了一个BlockingQueue，用来存储每一个提交进来的Runnable，同时，线程池内部会开启一些线程，来对Runnable队列进行消费处理。
而netty线程池里面，BlockingQueue不再是整个线程池全局唯一，而是每一个线程都会绑定一个BlockingQueue，当有Runnable提进来的时候，会有这样一个选择器将Runnable提交给目标线程的BlockingQueue。这样每个队列只有一个线程消费，一定程度上避免了锁抢占带来的overhead。
