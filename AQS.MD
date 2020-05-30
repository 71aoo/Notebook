# AQS(AbstractQueuedSynchronizer)

## Java层面如何实现一个锁

如何只使用 **Java** 去实现一个锁？

### 方式一

使用一个 **state** 属性来表示当前锁状态（**0**没有线程持有锁，**1**有线程持有锁），没有或得锁的线程一直在**自旋**（死循环，不断尝试获得锁，直到成功）。

伪代码：

```java
public class Lock {

    // 表示锁的状态，保证线程之间的可见性
    volatile int state = 0;

    public void lock(){

        // 直到获取锁成功       
        while (!compareAndSetState(0, 1)){

        }

        // 上锁逻辑

    }

    public void unLock(){

        // 判断是否是持有锁的线程
        
        // 修改状态
        state = 0;
    }

    private boolean compareAndSetState(int expect,int update){

        //CAS 操作修改 state，成功返回 true，失败返回 false
    }
}
```

这样是可以实现一个锁，但是，如果一个线程获取不到锁，它也会一直在运行（获取锁），消耗CPU资源，造成浪费。

解决思路：**让获取不到锁的线程让出CPU资源。**

### 方式二

**自旋 + yield**

```
public class Lock {

    // 表示锁的状态，保证线程之间的可见性
    volatile int state = 0;

    public void lock(){

        while (compareAndSetState(0, 1)){

			// 让出CPU资源
			yield()
        }

        // 上锁逻辑

    }

    public void unLock(){

        state = 0;
    }

    private boolean compareAndSetState(int expect,int update){

        //CAS 操作修改 state，成功返回 true，失败返回 false
    }
}
```

实现一个 **yield** 方法，让获取不到锁的线程让出CPU，但是，线程数少还好，如果是上百上千个线程，不断让出CPU，然后又不断竞争，这样还是会在消耗CPU资源。

**如果让没有获得锁的线程睡一段时间，再醒来竞争呢？**

### 方式三

**sleep + 自旋**

```java
public class Lock {

    // 表示锁的状态，保证线程之间的可见性
    volatile int state = 0;

    public void lock(){

        while (compareAndSetState(0, 1)){

			// 让出CPU资源,并沉睡
			sleep(n)
        }

        // 上锁逻辑

    }

    public void unLock(){

        state = 0;
    }

    private boolean compareAndSetState(int expect,int update){

        //CAS 操作修改 state，成功返回 true，失败返回 false
    }
}
```

sleep 是可以 sleep，但是sleep多久呢，这个恐怕没有人知道，时间长了，效率低，时间短了，消耗资源。这种方式还是差了点意思。

**是否可以让未获得锁的线程进入等待，等待持有锁的线程执行完，它们再来竞争锁**

### 方式四

**park + 自旋**

```java
public class Lock {

    // 表示锁的状态，保证线程之间的可见性
    volatile int state = 0;
    
    // 等待队列，未获的锁的线程进入等待队列
    Queue parkQueue;

    public void lock(){

        while (compareAndSetState(0, 1)){
			
            // 进入等待队列逻辑
			park();
        }

        // 上锁逻辑

    }

    public void unLock(){
        
        //修改锁状态
		
 		// 唤醒等待队列线程
        t = parkQueue;
        
        unPark(t);
    }

    private boolean compareAndSetState(int expect,int update){

        //CAS 操作修改 state，成功返回 true，失败返回 false
    }
    
    private void park(){
        
        // 未获得锁，加入等待队列
        parkQueue.add(Thread.currentThread());
        
        // 释放CPU资源
        releaseCPU();
          
    }
    
    private void unPark(Queue parkQueue){
        
        // 唤醒等待队列
    }
}
```

让未获取锁的线程进入等待队列并释放CPU，直到持有锁的线程释放锁，再唤醒等待队列里的线程。

在 **java.util.concurrent.locks** 包下的 **AbstractQueuedSynchronizer** 抽象类就使用这种方式实现的。下面将研究其具体实现逻辑。



## AbstractQueuedSynchronizer

### 简介

**AQS** 是 **java.util.concurrent.locks** 包一个抽象类，它实现了对**同步状态的管理**，以及对**阻塞线程进行排队**，**等待通知**等等一些底层的实现处理，同时它还包括的以下几个核心：**同步队列，独占式锁的获取和释放，共享锁的获取和释放以及可中断锁，超时等待锁获取这些特性的实现**。

### 加锁

看看AQS如何获得锁，和如何处理未获得锁的线程。



#### acquire(int arg)

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**if** 判断，通过 **tryAcquire** 方法去尝试获得锁（名字取得讲究，尝试获取），猜测获得锁失败，**tryAcquire** 返回 **false**，那么 **!tryAcquire** 就为 **true**，进入 **addWaiter** 方法，把当前线程包装为一个 **Node** 节点，**acquireQueued** 再把包装好的 **Node** 加入队列；如果 **tryAcquire** 返回 **true**，则获取锁成功。

具体查看源码便知。



#### tryAcquire(int acquires)

```JAVA
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

**AQS** 里面没有具体实现 **tryAcquire** 方法，因为 **AQS** 是一个抽象类，可以理解为一个抽象锁，具体哪种锁，加锁的方式是不一样的，交由子类重写，针对锁的不同，实现具体加锁逻辑。



#### addWaiter(Node mode)

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

分析

```java
addWaiter(Node.EXCLUSIVE);
    
// 初始值为空
static final Node EXCLUSIVE = null;    
```

该方法时获取锁失败后，把线程包装成一个 **Node** 节点的方法，**acquire** 方法中传入的是 **Node.EXCLUSIVE**。

```java
Node node = new Node(Thread.currentThread(), mode);


Node(Thread thread, Node mode) {     // Used by addWaiter
    this.nextWaiter = mode;
    this.thread = thread;
}
```

进入 **addWaiter** 方法，先新建个 **Node** 包装当前线程

```
Node pred = tail;
if (pred != null) {
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
        pred.next = node;
        return node;
    }
}

// 队列头部
private transient volatile Node tail;

// 队列尾部
private transient volatile Node head;
```

**AQS** 记录着队列头节点（**head**）和尾节点（**tail**）。拿到队列的尾节点，如果尾部不为空，那么说明该线程前面还有等待线程，用 **CAS** 进行尾插法操作，插入新的节点，如果为空，说明队列刚初始化，直接把包装节点传入 **enq** 方法，同时这里 **CAS** 尾插不成功也会进入 **enq**  方法。最后都是返回新建 **Node**。



#### enq(final Node node) 

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

看到 **for (;;)** ，很明显这里是个自旋，主要有两步操作。

第一次自旋，尾部为空，说明队列没有初始化，新建了一个空节点，设置头部和尾部，再循环一次，这时候尾部已经不为空，再用 **CAS** 尾插法把新建节点设置为尾节点，不断重试知道成功。

如果是 **addWaiter** 方法中尾插失败，则 **enq** 方法自旋尾插入，知道成功返回尾部。



#### 队列结构

看完 **addWaiter** 和 **enq** 方法，基本可以了解队列的结果，**就是个链表**。

**初始化**

先创建一个包装当前线程的 **Node**

![image-20200527170424368](https://gitee.com/Playwi0/MyImage/raw/master/NoteImage/image-20200527170424368.png)

再初始化一个空的节点，指向 **head** 属性，然后与新建节点进行链接，

![image-20200527171441726](https://gitee.com/Playwi0/MyImage/raw/master/NoteImage/image-20200527171441726.png)



**队列存在**

新建节点直尾接插入该队列

![image-20200527172013275](https://gitee.com/Playwi0/MyImage/raw/master/NoteImage/image-20200527172013275.png)





#### acquireQueued(final Node node, int arg)

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在进入 **acquireQueued** 之前，调用 **addWaiter** 包装线程，并插入队列，再将包装好的 **Node** 传入到 **acquireQueued** 方法。

```java
boolean failed = true;
boolean interrupted = false;
```

两个标记，作用未知

```java
final Node p = node.predecessor();


final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
        throw new NullPointerException();
    else
        return p;
}
```

调用 **node** 内部方法，获得前驱节点 

```java
if (p == head && tryAcquire(arg)) {
    setHead(node);
    p.next = null; // help GC
    failed = false;
    return interrupted;
}
```

前面介绍了同步队列的结构，第一个节点是空的（可能用来表示持有锁的线程）。这里 p 表示当前节点（线程）的前驱，如果他的前驱是队列的头部，说明当前节点在队列的第二位（前面是持有锁的线程），有资格尝试获得锁。按理说进入到 **acquireQueued**  是因为线程第一次尝试获得锁（tryAcquire）失败，包装成 node 节点去排队，为什么再这里还有再次尝试获得锁？可能是在插入队列的时候，锁已经释放了，这里再次尝试获得锁，避免当前线程进入排队。

```java
 private void setHead(Node node) {
     head = node;
     node.thread = null;
     node.prev = null;
 }
```

如果获得锁成功，把当前节点设置尾头节点，代替原来头节点的位置。改变标记，返回当前节点是否被打断标记。

```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
```

如果当前节点的前驱节点不是头节点，说明前面还有线程 node 在排队，应该进入阻塞逻辑。

一般进入 **acquireQueued** 方法的节点，绝大部分都是要进行阻塞处理，由两个方法实现 **shouldParkAfterFailedAcquire** 和 **parkAndCheckInterrupt**



#### shouldParkAfterFailedAcquire(Node pred, Node node)

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
        
    } else {
        
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    
   return false;
}
```

分析

```java
int ws = pred.waitStatus;
```

获取前驱节点状态（排队还是已获得锁）

```
if (ws == Node.SIGNAL)
	return true;
```

如果当前节点和前驱节点状态一样（阻塞），返回 **true**，表示当前线程可以安心休息。

```java
if (ws > 0) {
    do {
        node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;

} 
```

如果不和前驱节点的状态一样，说明前驱节点有可能已经放弃竞争，等死的节点，**do while** 循环，直到获取到一个正常的节点（pred.waitStatus <= 0，小于 0 阻塞，等于 0 在竞争）。

```java
else {

    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
}
```

如果前驱节点正常，把前驱节点设置为阻塞状态，这里有可能失败，前驱节点有可能刚好正在竞争锁。

```java
return false;
```

除了前驱节点处于等待状态，否则都会返回失败，再进入 **acquireQueued**  的 **for (;;)** 自旋获得锁或者，再次进入排队。



#### parkAndCheckInterrupt()

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

前面源码可以看到，**shouldParkAfterFailedAcquire** 方法主要是改变节点为阻塞状态，直到成功或者节点成功抢占锁。那么 **parkAndCheckInterrupt** 方法在 **shouldParkAfterFailedAcquire** 之后执行，很明显节点状态已经设置好，走到这里说明线程要进入排队了。

```java
LockSupport.park(this);



public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```

先调用 **LockSupport** 类中的 **park** 方法让线程进入阻塞状态（内部逻辑是有 **native** 方法实现）。

```java
return Thread.interrupted();
```

返回线程是否被打断标。



#### 总结

前面所有分析，否是加锁部分逻辑。

- 第一次拿锁，加锁（**tryAcquire**）成功则不需要进入排队，
- 第一次拿锁失败，队列未初始化，则初始化队列（头节点为空的链表），然后包装成节点插入队列，这时应该在队列第二位，有资格再一次竞争锁，再次失败则进入排队，成功直接获得锁。
- 第一次拿锁失败，队列已初始化，包装成节点进入队列排队，前驱节点是阻塞状态，则自己安心阻塞，如果前面线程已放弃竞争，则自己位置前进以为，不断重复这个过程，直到找到正常节点，改变状态。

### 释放锁

有加锁肯定就有释放锁，当然释放锁不可能是简单改变锁的状态，还要负责唤醒正在等待的线程 



#### release(int arg)

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

调用 **tryRelease** 方式尝试释放锁。

```java
 Node h = head;
```

如果成功，则获取队列头部

```java
if (h != null && h.waitStatus != 0)
    unparkSuccessor(h);
return true;
```

如果队列不为空，且正在阻塞状态，则调用  **unparkSuccessor** 唤醒队列。最终返回 **true**

```java
 return false;
```

释放锁失败，返回 **false**



#### tryRelease(int arg)

```java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

同样的，**tryRelease**方法的具体实现交给具体锁。



####  unparkSuccessor(Node node)

```java
private void unparkSuccessor(Node node) {
  
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

前面传进来是头部，一般是处于阻塞状态（加锁时 **shouldParkAfterFailedAcquire** 方法设置的）。

```java
int ws = node.waitStatus;
if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
```

先拿到节点状态，如果时阻塞状态，则改变为正常状态 0。

```java
Node s = node.next;
```

头部为空节点，拿到下一个节点（真正在排队节点）。

```java
if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
            s = t;
}
```

如果下一个节点是空节点或着已经放弃竞争，则从队列尾部开始查找有效节点（**waitStatus <= 0**），如果找到正常节点或者下一个是阻塞节点,，则进入唤醒逻辑

```java
if (s != null)
    LockSupport.unpark(s.thread);


public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

这里是防止从尾部开始查找节点已经为空或者说来就是一个空队列。



#### 总结

解锁逻辑相对加锁逻辑比较简单，主要就是改变锁的状态，和唤醒队列。



### AQS总结

结合前面加锁和解锁分析，可以解决几个疑问。

为什么头节点为空？

从 **shouldParkAfterFailedAcquire** 方法和 **unparkSuccessor**，可以推测，某个节点的前驱节点是表示这个节点的状态，所以在加锁的时候要保证前驱节点的状态，解锁的也要改变前驱节点状态，如果前驱节点状态都错误，可能后面都错误，所以从尾部开始查找，查看有没有正常节点。

当然 **AQS** 不仅仅只有这几个方法，还有共享锁加解锁逻辑等。有兴趣可以继续阅读



## 公平锁（FairSync）

**FairSync** 是 **AQS** 子类之一（间接），**AQS** 是抽象锁，那么 **FairSync** 就是具体锁，公平锁的一种。

**AQS** 留下了两个主要的抽象方法：**tryAcquire** 和 **tryRelease** ，查看这个两个方法的具体实现，来看锁的具体实现原理。

### 加锁

公平锁加锁



#### tryAcquire(int acquires)

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

先获取当前线程

```java
final Thread current = Thread.currentThread();
```

再获得当前锁的状态，是否被其他线程上锁。

```java
int c = getState();

// Method
protected final int getState() {
    return state;
}

//  属性，0 表示未有线程持有锁，1 表示已被线程持有锁
private volatile int state;
```

如果当前线程是第一个到这来拿取锁的，那么 **c = 0**，则进入下面逻辑

```java
if (c == 0) {
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

**hasQueuedPredecessors** 方法判断是否有前驱节点，也就是是否需要排队，需要返回 **true** ，不需要返回 **false** ，第一次上锁队列都是没初始化，如果不需要排队就 **CAS** 操作更改锁的状态为上锁状态。

```java
setExclusiveOwnerThread(current);


protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}


private transient Thread exclusiveOwnerThread;
```

上锁完后要记录当前持有锁的线程，因为要为下面重入作铺垫。

```java
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```

如果锁已经被某个线程持有，那么 **c = 1**，还有一种可能可以拿到锁，那就是锁的重入，就是当前线程在锁的同步块里面加锁，又是同一把锁，那么这种情况可以拿到锁。

```java
current == getExclusiveOwnerThread()
```

比对持有锁的线程和当前线程是否是同一个线程，是就是重入锁。

```java
nt nextc = c + acquires;
f (nextc < 0)
    throw new Error("Maximum lock count exceeded");
setState(nextc);
```

记录加锁次数，解锁次数等于加锁次数才算解锁成功。这里返回 **true**



#### hasQueuedPredecessors()

```java
public final boolean hasQueuedPredecessors() {

    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

这个方法不简单，公平锁的核心所在。看看如何保证公平。

```java
h != t
```

如果是相等的，有两种情况：

- **head** 和 **tail** 都为空，说名明队列没有初始化，说明当前线程是第一个来拿锁的，则可以直接拿到锁。
- **head** 和 **tail** 都指向同一个节点，这种情况只有一种可能，根据队列结构可以判断，是空的头节点；这种情况说明队列虽然初始化，但是是空的队列，没有人在排队。可以直接拿到锁。

如果不相等，说明队列已经初始化，并且有可能有节点在排队。

```java
((s = h.next) == null || s.thread != Thread.currentThread())
```

队列已初始化，头节点为空，真正在排队是第二个节点。这时有两种情况：

- 第二个节点是空节点，这种情况基本不可能出现。
- 第二个节点是不是空节点，继续判断第二个节点里面的线程是否为当前线程，如果是，说明你是排在最前面哪一个，可以去拿锁；如果不是，说明队列有线程排在当前线程前面，当前线程需要去排队。



### 解锁

公平锁的解锁逻辑



#### tryRelease(int releases)

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

分析

```java
int c = getState() - releases;
```

先做好解锁标记

```java
if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
```

如果持有锁的线程不是当前线程，那么说明没有拿到锁就解锁，捣乱。

```java
boolean free = false;
if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
}
setState(c);
return free;
```

如果 **c = 0**，说明当前线程全部释放锁（包括重入锁），则可以把锁设置为无锁状态，如果 **c != 0**，说明只是释放了一个重入锁，锁并没有完全释放 ，还是返回 **false**，不会去唤醒队列。



### 总结

经过分析，公平锁的原则就是，谁先来拿锁，谁就排在前面，谁就优先，先到先得。即使有线程来的是刚好没有线程持有锁（可能线程刚好释放），也要先判断队列是否有线程排队，有则当前线程要靠后，这就是公平锁。



## 非公平锁（NonfairSync）

**NonfairSync** 也是 **AQS** 的子类之一，是一种非公平锁，同样的，主要查看它的 **tryAcquire** 和 **tryRelease** 方法。



### 加锁

 非公平锁加锁逻辑



#### tryAcquire(int acquires)

```
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

这里非公平锁自定义了一个方法 **nonfairTryAcquire**



#### nonfairTryAcquire(int acquires)

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

整体看，这里代码和公平锁的代码相差不大

```java
if (c == 0) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

这一步少了一个 **hasQueuedPredecessors**  方法，改方法主要是在线程拿锁前，判断同步队列里面是否还有等待线程，而这里去掉，说明无论同步队列有没有等待线程，都可以去抢占锁。也就是说，线程来拿锁，无论先来还是后到，只有拿不到锁才回去排队。



### 解锁

非公平锁释放锁逻辑



#### tryRelease(int releases)

非公平锁释放锁和公平锁释放共用一个方法，两个锁释放锁都是这个方法具体看前面。



### 总结

结合公平锁来看

```java
// 公平锁
final void lock() {
    acquire(1);
}


// 非公平锁
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

公平锁有先来后到原则，而非公平锁拿到就是你的，拿不到就去排队。





