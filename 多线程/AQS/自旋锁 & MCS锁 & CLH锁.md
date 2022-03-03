## 1 自旋锁

以 synchronized 为代表的阻塞同步，因为阻塞线程和恢复线程的操作都需要涉及到操作系统层面的用户态和内核态之间的切换，这对系统的性能影响很大。

自旋锁的策略是当线程去获取一个锁时，如果发现该锁已经被其它线程占有，那么它不马上放弃 CPU 的执行时间片，而是进入一个“无意义”的循环，查看该线程是否已经放弃了锁。

## 1.1 使用场景

但自旋锁适用于**临界区**比较小的情况，如果锁持有的时间过长，那么自旋操作本身就会白白耗掉系统的性能。

## 1.2 简单实现

```java
public class SpinLock {
   private AtomicReference<Thread> owner = new AtomicReference<Thread>();
   public void lock() {
       Thread currentThread = Thread.currentThread();
        // 如果锁未被占用，则设置当前线程为锁的拥有者
       while (!owner.compareAndSet(null, currentThread)) {}
   }

   public void unlock() {
       Thread currentThread = Thread.currentThread();
        // 只有锁的拥有者才能释放锁
       owner.compareAndSet(currentThread, null);
   }
}
```

这种自旋锁有一些缺点：

+ 没有保证公平性，等待获取锁的线程之间，无法按先后顺序分别获得锁

+ 由于多个线程会去操作同一个变量 owner，在 CPU 的系统中，存在着各个 CPU 之间的缓存数据需要同步，保证一致性，这会带来性能问题。

## 1.3 公平自旋锁实现

让每个锁拥有一个服务号，表示正在服务的线程，而每个线程尝试获取锁之前需要先获取一个排队号，然后不断轮询当前锁的服务号是否是自己的服务号，如果是，则表示获得了锁，否则就继续轮询

```java
public class TicketLock {
   private AtomicInteger serviceNum = new AtomicInteger(); // 服务号
   private AtomicInteger ticketNum = new AtomicInteger(); // 排队号

   public int lock() {
       // 首先原子性地获得一个排队号
       int myTicketNum = ticketNum.getAndIncrement();
       // 只要当前服务号不是自己的就不断轮询
       while (serviceNum.get() != myTicketNum) {
       }
       return myTicketNum;
    }
  
    public void unlock(int myTicket) {
        // 只有当前线程拥有者才能释放锁
        int next = myTicket + 1;
        serviceNum.compareAndSet(myTicket, next);
    }
}
```

然存在前面说的多 CPU 缓存的同步问题，因为**每个线程占用的 CPU 都在同时读写同一个变量 serviceNum**，这会导致繁重的系统总线流量和内存操作次数，从而降低了系统整体的性能。

# 2 MCS自旋锁 

MCS 的名称来自其发明人的名字：John Mellor-Crummey和Michael Scott。
MCS 的实现是基于链表的，每个申请锁的线程都是链表上的一个节点，这些线程会一直轮询自己的本地变量，来知道它自己是否获得了锁。已经获得了锁的线程在释放锁的时候，负责通知其它线程，这样 CPU 之间缓存的同步操作就减少了很多，仅在线程通知另外一个线程的时候发生，降低了系统总线和内存的开销。实现如下所示：

```java
public class MCSLock {
    public static class MCSNode {
        volatile MCSNode next;
        volatile boolean isWaiting = true; // 默认是在等待锁
    }
    volatile MCSNode queue;// 指向最后一个申请锁的MCSNode
    private static final AtomicReferenceFieldUpdater<MCSLock, MCSNode> UPDATER = AtomicReferenceFieldUpdater.newUpdater(MCSLock.class, MCSNode.class, "queue");

    public void lock(MCSNode currentThread) {
        MCSNode predecessor = UPDATER.getAndSet(this, currentThread);// step 1
        if (predecessor != null) {
            predecessor.next = currentThread;// step 2
            while (currentThread.isWaiting) {// step 3
            }
        } else { // 只有一个线程在使用锁，没有前驱来通知它，所以得自己标记自己已获得锁
            currentThread.isWaiting = false;
        }
    }

    public void unlock(MCSNode currentThread) {
        if (currentThread.isWaiting) {// 锁拥有者进行释放锁才有意义
            return;
        }

        if (currentThread.next == null) {// 检查是否有人排在自己后面
            if (UPDATER.compareAndSet(this, currentThread, null)) {// step 4
                // compareAndSet返回true表示确实没有人排在自己后面
                return;
            } else {
                // 突然有人排在自己后面了，可能还不知道是谁，下面是等待后续者
                // 这里之所以要忙等是因为：step 1执行完后，step 2可能还没执行完
                while (currentThread.next == null) { // step 5
                }
            }
        }
        currentThread.next.isWaiting = false;
        currentThread.next = null;// for GC
    }
}
```



# 3 CLH 自旋锁

**CLH 锁与 MCS 锁最大的不同是，MCS 轮询的是当前队列节点的变量，而 CLH 轮询的是当前节点的前驱节点的变量，来判断前一个线程是否释放了锁。**

```java
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;
public class CLHLock {
    public static class CLHNode {
        private volatile boolean isWaiting = true; // 默认是在等待锁
    }
    private volatile CLHNode tail ;
    private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> UPDATER = AtomicReferenceFieldUpdater
            . newUpdater(CLHLock.class, CLHNode .class , "tail" );
    public void lock(CLHNode currentThread) {
        CLHNode preNode = UPDATER.getAndSet( this, currentThread);
        if(preNode != null) {//已有线程占用了锁，进入自旋
            while(preNode.isWaiting ) {
            }
        }
    }

    public void unlock(CLHNode currentThread) {
        // 如果队列里只有当前线程，则释放对当前线程的引用（for GC）。
        if (!UPDATER .compareAndSet(this, currentThread, null)) {
            // 还有后续线程
            currentThread.isWaiting = false ;// 改变状态，让后续线程结束自旋
        }
    }
}
```

## 3.1 MCS和CLH差异

1. 从自旋的条件来看，CLH是在前驱节点的属性上自旋，而MCS是在本地属性变量上自旋。
2. 从链表队列来看，CLH的队列是隐式的，CLHNode并不实际持有下一个节点；MCS的队列是物理存在的。
3. CLH锁释放时只需要改变自己的属性，MCS锁释放则需要改变后继节点的属性。

## 4.关于CLH与MCS的思考

CLH在SMP系统结构下该法是非常有效的。但在NUMA系统结构下，每个线程有自己的内存，如果前趋结点的内存位置比较远，自旋判断前趋结点的locked域，性能将大打折扣，一种解决NUMA系统结构的思路是MCS队列锁。

**为什么AQS不用MCS队列实现呢？**
个人想法是：AQS在入队后自旋次数并不是很多，线程会被挂起，所以在前驱节点自旋还是在自身节点自旋区别没有那么大，而释放锁的时候，CLH只需要改变自己的属性，而MCS需要改变后继节点的属性，反而在NUMA架构下CLH更占优势。此外CLHNode并不实际持有下一个节点，而MCS队列是物理存在的。内存占用上CLH也更优。所以AQS选用CLH队列实现

