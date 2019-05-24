---
title: AbstractQueuedSynchronizer解析
date: 2018-07-02 15:39:56
tags:
    - java
    - 代码
categories:
    - Java并发
---

#### AbstractQueuedSynchronizer 数据结构

```java
 	/** 
     * Head of the wait queue , lazily initialized . Except for 
     * initialization , it is modified only via method setHead .   Note : 
     * If head exists , its waitStatus is guaranteed not to be 
     * CANCELLED . 
     */ 
    private transient volatile Node head; 

    /** 
     * Tail of the wait queue , lazily initialized . Modified only via 
     * method enq to add new wait node . 
     */ 
    private transient volatile Node tail; 

    /** 
     * The synchronization state . 
     */ 
    private volatile int state;
```

稍微注意下在线程争用锁是才会初始化链表



#### **AbstractQueuedSynchronizer.Node** 数据结构

```java
    /** 
    * Status field , taking on only the values : 
    *    SIGNAL :      The successor of this node is ( or will soon be ) 
    *                blocked ( via park ), so the current node must 
    *                unpark its successor when it releases or 
    *                cancels . To avoid races , acquire methods must 
    *                first indicate they need a signal , 
    *                then retry the atomic acquire , and then , 
    *                on failure , block . 
    *    CANCELLED :   This node is cancelled due to timeout or interrupt . 
    *                Nodes never leave this state . In particular , 
    *                a thread with cancelled node never again blocks . 
    *    CONDITION :   This node is currently on a condition queue . 
    *                It will not be used as a sync queue node 
    *                until transferred , at which time the status 
    *                will be set to 0. ( Use of this value here has 
    *                nothing to do with the other uses of the 
    *                field , but simplifies mechanics .) 
    *    PROPAGATE :   A releaseShared should be propagated to other 
    *                nodes . This is set ( for head node only ) in 
    *                doReleaseShared to ensure propagation 
    *                continues , even if other operations have 
    *                since intervened . 
    *    0:           None of the above 
    * 
    * The values are arranged numerically to simplify use . 
    * Non - negative values mean that a node doesn ' t need to 
    * signal . So , most code doesn ' t need to check for particular 
    * values , just for sign . 
    * 
    * The field is initialized to 0 for normal sync nodes , and 
    * CONDITION for condition nodes .   It is modified using CAS 
    * ( or when possible , unconditional volatile writes ). 
    */ 
    volatile int waitStatus; 

    /** 
    * Link to predecessor node that current node / thread relies on 
    * for checking waitStatus . Assigned during enqueuing , and nulled 
    * out ( for sake of GC ) only upon dequeuing .   Also , upon 
    * cancellation of a predecessor , we short - circuit while 
    * finding a non - cancelled one , which will always exist 
    * because the head node is never cancelled : A node becomes 
    * head only as a result of successful acquire . A 
    * cancelled thread never succeeds in acquiring , and a thread only 
    * cancels itself , not any other node . 
    */ 
    volatile Node prev; 

    /** 
    * Link to the successor node that the current node / thread 
    * unparks upon release . Assigned during enqueuing , adjusted 
    * when bypassing cancelled predecessors , and nulled out ( for 
    * sake of GC ) when dequeued .   The enq operation does not 
    * assign next field of a predecessor until after attachment , 
    * so seeing a null next field does not necessarily mean that 
    * node is at end of queue . However , if a next field appears 
    * to be null , we can scan prev ' s from the tail to 
    * double - check .   The next field of cancelled nodes is set to 
    * point to the node itself instead of null , to make life 
    * easier for isOnSyncQueue . 
    */ 
    volatile Node next; 

    /** 
    * The thread that enqueued this node . Initialized on 
    * construction and nulled out after use . 
    */ 
    volatile Thread thread; 

    /** 
    * Link to next node waiting on condition , or the special 
    * value SHARED . Because condition queues are accessed only 
    * when holding in exclusive mode , we just need a simple 
    * linked queue to hold nodes while they are waiting on 
    * conditions . They are then transferred to the queue to 
    * re - acquire . And because conditions can only be exclusive , 
    * we save a field by using special value to indicate shared 
    * mode . 
    */ 
    Node nextWaiter;
```



#### AbstractQueuedSynchronizer** 的数据结构（盗用的图）

![](https://i.loli.net/2019/05/20/5ce25d1a4577394145.png)

 

#### **AbstractQueuedSynchronizer** **做了什么** ?

内部维护state和CLH队列，负责在资源争用时线程入队，资源释放时唤醒队列中线程。

而实现类只需要实现 **什么条件获取资源成功** 和 **什么条件释放资源** 成功就可以了

所以，最简单的CountDownLatch使用AbstractQueuedSynchronizer实现非常简单：

-              申明AbstractQueuedSynchronizer的state数量（比如十个）
-              await方法尝试获取资源，如果state>0表示获取失败（ **什么条件获取资源成功** ，CountDownLatch实现），获取失败线程休眠（AbstractQueuedSynchronizer负责）
-             countDown方法state-1，如果state==0表示资源释放成功( **什么条件释放资源成功** ，CountDownLatch实现)，唤醒队列中所有线程（AbstractQueuedSynchronizer负责）



#### **AbstractQueuedSynchronizer** 怎么做的?

顺着ReentrantLock lock、unlock看一遍我们就大致总结出AbstractQueuedSynchronizer工作原理了

先简单介绍下ReentrantLock特性：可重入，中断，有超时机制。



#### **ReentrantLock lock()** **流程** **(** **再盗图** )

![](https://i.loli.net/2019/05/20/5ce25da81509c52443.png)

 

黄色表示ReentrantLock实现，绿色表示AbstractQueuedSynchronizer内部实现

1. lock方法入口 直接调用 [AbstractQueuedSynchronizer.acquire方法](http://wiki.lianjia.com/display/JTZPCP/AQS####AbstractQueuedSynchronizer.acquire)
2. tryAcquire
3. addWaiter
4. acquireQueued



##### AbstractQueuedSynchronizer.acquire

```java
**public** final void acquire ( int arg) { 
    **if** (! tryAcquire (arg) && 
        acquireQueued ( addWaiter ( Node . EXCLUSIVE ), arg)) 
        selfInterrupt (); 
}
```

获取的锁的逻辑：直接获取成功则返回，如果没有获取成功入队休眠（对就是这么简单）

下面我们仔细一个一个方法看

##### ReentrantLock.tryAcquire

我这里贴的时非公平的所获取，公平和不公平的区别在于公平锁老老实实的会进入队列排队，非公平锁会先检查资源是否可用，如果可用不管队列中的情况直接尝试获取锁。

```java
final boolean nonfairTryAcquire ( int acquires) { 
    final Thread current = Thread . currentThread (); 
    int c = getState (); 
    if (c == 0 ) { 
        if ( compareAndSetState ( 0 , acquires)) { 
            setExclusiveOwnerThread (current); 
            return true ; 
        } 
    } 
    else if (current == getExclusiveOwnerThread ()) { 
        int nextc = c + acquires; 
        if (nextc < 0 ) // overflow
            throw new Error ( "Maximum lock count exceeded" ); 
        setState (nextc); 
        return true ; 
    } 
    return false ; 
}
```

ReentrantLock.tryAcquire读取到state==0时尝试占用锁，并保证同一线程可以重复占用。其他情况下获取资源失败。如果获取成功就没啥事了，不过关键不就是锁争用的时候是如何处理的吗？

##### AbstractQueuedSynchronizer.addWaiter(Node.EXCLUSIVE)

```java
private Node addWaiter ( Node mode) { 
    Node node = new Node (mode); 

    for (;;) { 
        Node oldTail = tail; 
        if (oldTail != null ) { 
            node. setPrevRelaxed (oldTail); 
            if ( compareAndSetTail (oldTail, node)) { 
                oldTail. next = node; 
                return node; 
            } 
        } else { 
            initializeSyncQueue (); 
        } 
    } 
}
```

一旦锁争用，一定会初始化队列（因为排队的线程需要前驱节点唤醒，所以要初始化一个前驱节点），之后自旋成为队列尾节点。

简单来说就是获取不到锁就放进队列里维护起来，等锁释放的时候再用。

这里还有一个 **很具有参考性的小细节** ：先设置新节点的前驱结点，自旋成为尾节点后设置前驱的后驱

##### AbstractQueuedSynchronizer.acquireQueued

```java
final boolean acquireQueued ( final Node node, int arg) { 
    boolean interrupted = false ; 
    try { 
        for (;;) { 
            final Node p = node. predecessor (); 
            if (p == head && tryAcquire (arg)) { 
                setHead (node); 
                p. next = null ; // help GC 
                return interrupted; 
            } 
            if ( shouldParkAfterFailedAcquire (p, node)) 
                interrupted |= parkAndCheckInterrupt (); 
        } 
    } catch ( Throwable t) { 
        cancelAcquire (node); 
        if (interrupted) 
            selfInterrupt (); 
        throw t; 
    } 
} 

private static boolean shouldParkAfterFailedAcquire ( Node pred, Node node) { 
    int ws = pred. waitStatus ; 
    if (ws == Node . SIGNAL ) 
        /* 
             * This node has already set status asking a release 
             * to signal it, so it can safely park. 
             */ 
        return true ; 
    if (ws > 0 ) { 
        /* 
             * Predecessor was cancelled. Skip over predecessors and 
             * indicate retry. 
             */ 
        do { 
            node. prev = pred = pred. prev ; 
        } while (pred. waitStatus > 0 ); 
        pred. next = node; 
    } else { 
        /* 
             * waitStatus must be 0 or PROPAGATE.  Indicate that we 
             * need a signal, but don't park yet.  Caller will need to 
             * retry to make sure it cannot acquire before parking. 
             */ 
        pred. compareAndSetWaitStatus (ws, Node . SIGNAL ); 
    } 
    return false ; 
} 

private final boolean parkAndCheckInterrupt () { 
        LockSupport . park ( this ); 
        return Thread . interrupted (); 
    }
```

前面只是维护下链表数据结构，这里负责找到合适的唤醒前驱，然后让线程休眠。

这里主要是一个循环过程：

1. 检查是否能获取到锁，获取到则返回
2. 失败则寻找前面最近的未放弃争用的前驱，把前驱的waitStatus设置为-1，并把放弃争用的节点抛弃
3. 检查是否能休眠
4. 使用Usafe.park休眠（不是wait）

 

#### **ReentrantLock lock** 总结

![](https://i.loli.net/2019/05/20/5ce25ea9eeba265273.png)

 

#### ReentrantLock unlock()

```java
public final boolean release ( int arg) { 
    if ( tryRelease (arg)) { 
        Node h = head; 
        if (h != null && h. waitStatus != 0 ) 
            unparkSuccessor (h); 
        return true ; 
    } 
    return false ; 
} 

protected final boolean tryRelease ( int releases) { 
    int c = getState () - releases; 
    if ( Thread . currentThread () != getExclusiveOwnerThread ()) 
        throw new IllegalMonitorStateException (); 
    boolean free = false ; 
    if (c == 0 ) { 
        free = true ; 
        setExclusiveOwnerThread ( null ); 
    } 
    setState (c); 
    return free; 
} 

private void unparkSuccessor ( Node node) { 
    /* 
         * If status is negative (i.e., possibly needing signal) try 
         * to clear in anticipation of signalling.  It is OK if this 
         * fails or if status is changed by waiting thread. 
         */ 
    int ws = node. waitStatus ; 
    if (ws < 0 ) 
        node. compareAndSetWaitStatus (ws, 0 ); 

    /* 
         * Thread to unpark is held in successor, which is normally 
         * just the next node.  But if cancelled or apparently null, 
         * traverse backwards from tail to find the actual 
         * non-cancelled successor. 
         */ 
    Node s = node. next ; 
    if (s == null || s. waitStatus > 0 ) { 
        s = null ; 
        for ( Node p = tail; p != node && p != null ; p = p. prev ) 
            if (p. waitStatus <= 0 ) 
                s = p; 
    } 
    if (s != null ) 
        LockSupport . unpark (s. thread ); 
}
```

unlock的代码特别简单：

1. 每unlock一次state-1
2. state == 0 时资源成功释放
3. 如果释放成功，唤醒第二个节点
4. 如果第二个节点没引用或者放弃争用，从队尾开始寻找可以唤醒的线程