参考博客：

https://www.jianshu.com/p/d5e2e3513ba3

https://yq.aliyun.com/articles/647326

SynchronousQueue内部实现了两个类，**一个是TransferStack类，用于非公平模式；另外一个类是TransferQueue，用于公平模式。**

TransferStack和TransferQueue都继承`Transferer`这个类只定义了一个方法`transfer`，此方法可以既可以执行put也可以执行take操作。这两个操作被统一到了一个方法中，因为在`dual`数据结构中，put和take操作是对称的，所以相近的所有结点都可以被结合。使用`transfer`方法是从长远来看的，它相比分为两个几乎重复的部分来说更加容易理解。

**清除操作在队列和栈中以不同的方式完成。对于队列，当结点被取消时，我们总是可以在O(1)时间立刻删除它。但是如果它被固定在队尾，它就必须等待直到其他取消操作完成。对于栈来说，我们需要以O(n)时间遍历来确保能够删除这个结点，不过这个操作可以和其他访问栈的线程同时进行。**

##非公平模式实现`TransferStack`

```java
static final class TransferStack<E> extends Transferer<E> {
    /** 表示一个未匹配的消费者 */
    static final int REQUEST    = 0;
    /** 表示一个未匹配的生产者 */
    static final int DATA       = 1;
    /** 匹配状态，代表生产者正在给消费者数据 */
    static final int FULFILLING = 2;
  
    static final class SNode {
        volatile SNode next;        // 栈中的下一个结点
        volatile SNode match;       // 匹配此结点的结点
        volatile Thread waiter;     // 节点对应的线程，控制 park/unpark
        Object item;                // 数据
        int mode;										// 节点类型
```

###核心方法`transfer`

基础算法，循环尝试下面三种操作中的一个：

1. 如果头节点为空或者已经包含了相同模式的结点，尝试将结点增加到栈中并且等待匹配。如果被取消，返回null

2. 如果头节点是一个模式不同的结点，尝试将一个`fulfilling`结点加入到栈中，匹配相应的等待结点，然后一起从栈中弹出，并且返回匹配的元素。匹配和弹出操作可能无法进行，由于其他线程正在执行操作3

3. 如果栈顶已经有了一个`fulfilling`结点，帮助它完成它的匹配和弹出操作，然后继续。

```java
E transfer(E e, boolean timed, long nanos) {
    SNode s = null; // constructed/reused as needed
    // 传入参数为null代表请求获取一个元素，否则表示插入元素
    int mode = (e == null) ? REQUEST : DATA;

    for (;;) {
        SNode h = head;
        // 操作1==>如果头节点为空或者和当前模式相同
        if (h == null || h.mode == mode) {
            // 设置超时时间为 0，立刻返回
            if (timed && nanos <= 0L) {
                if (h != null && h.isCancelled())
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null;
            // 构造一个结点并且设为头节点
            } else if (casHead(h, s = snode(s, e, h, mode))) {
                // 等待满足
                SNode m = awaitFulfill(s, timed, nanos);
                if (m == s) {               // wait was cancelled
                    clean(s);
                    return null;
                }
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item);
            }
        // 操作2==>检查头节点是否为FULFILLIING
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            if (h.isCancelled())            // already cancelled
                casHead(h, h.next);         // pop and retry
            // 更新头节点为自己
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                // 循环直到匹配成功
                for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m is s's match
                    if (m == null) {        // all waiters are gone
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink
                }
            }
        // 操作3==>帮助满足的结点匹配
        } else {                            // help a fulfiller
            SNode m = h.next;               // m is h's match
            if (m == null)                  // waiter is gone
                casHead(h, null);           // pop fulfilling node
            else {
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink
            }
        }
    }
}
```

当一个结点插入到栈中，它要么能和其他结点匹配然后一起出栈，否则就需要等待一个匹配的结点到来。在等待的过程中，一般使用自旋等待代替阻塞（在多处理器环境下），因为很有可能会有相应结点到来。如果自旋结束还没有匹配，那么就设置waiter然后阻塞自己，在阻塞自己之前还会再检查至少一次是否有匹配的结点。

如果等待的过程中由于超时到期或者中断，那么需要取消此节点，方法是将`match`字段指向自己，然后返回。

```java
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = shouldSpin(s)
        ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS)
        : 0;
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel();
        SNode m = s.match;
        if (m != null)
            return m;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel();
                continue;
            }
        }
        if (spins > 0) {
            Thread.onSpinWait();
            spins = shouldSpin(s) ? (spins - 1) : 0;
        }
        else if (s.waiter == null)
            s.waiter = w; // establish waiter so can park next iter
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD)
            LockSupport.parkNanos(this, nanos);
    }
}
```

###清除

在最坏的情况我们需要遍历整个栈来删除节点s。如果有多个线程并发调用`clean`方法，我们不会知道其他线程可能已经删除了此节点。

```java
void clean(SNode s) {
    s.item = null;   // forget item
    s.waiter = null; // forget thread
    SNode past = s.next;
    if (past != null && past.isCancelled())
        past = past.next;

    // 删除头部被取消的节点
    SNode p;
    while ((p = head) != null && p != past && p.isCancelled())
        casHead(p, p.next);

    // 移除中间的节点
    while (p != null && p != past) {
        SNode n = p.next;
        if (n != null && n.isCancelled())
            p.casNext(n, n.next);
        else
            p = n;
    }
}
```

##公平模式实现`TransferQueue`

```java
static final class TransferQueue<E> extends Transferer<E> {
    static final class QNode {
        volatile QNode next;          // next node in queue
        volatile Object item;         // CAS'ed to or from null
        volatile Thread waiter;       // to control park/unpark
        final boolean isData;
```

###核心方法`transfer`

基础算法，循环尝试下面两种操作中的一个：

1. 如果队列为空或者头节点模式和自己的模式相同，尝试将自己增加到队列的等待者中，等待被满足或者被取消

2. 如果队列包含了在等待的节点，并且本次调用是与之模式匹配的调用，尝试通过CAS修改等待节点item字段然后将其出队

```java
E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // saw uninitialized value
            continue;                       // spin

        // 如果队列为空或者模式与头节点相同
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next;
            // 如果有其他线程修改了tail，进入下一循环重读
            if (t != tail)                  // inconsistent read
                continue;
            // 如果有其他线程修改了tail，尝试cas更新尾节点，进入下一循环重读
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);
                continue;
            }
            // 超时返回
            if (timed && nanos <= 0L)       // can't wait
                return null;
            // 构建一个新节点
            if (s == null)
                s = new QNode(e, isData);
            // 尝试CAS设置尾节点的next字段指向自己
            // 如果失败，重试
            if (!t.casNext(null, s))        // failed to link in
                continue;
      
            // cas设置当前节点为尾节点
            advanceTail(t, s);              // swing tail and wait
            // 等待匹配的节点
            Object x = awaitFulfill(s, e, timed, nanos);
            // 如果被取消，删除自己，返回null
            if (x == s) {                   // wait was cancelled
                clean(t, s);
                return null;
            }

            // 如果此节点没有被模式匹配的线程出队
            // 那么自己进行出队操作
            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // complementary-mode
            QNode m = h.next;               // node to fulfill
            // 数据不一致，重读
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // m already fulfilled     m已经匹配成功了
                x == m ||                   // m cancelled             m被取消了
                !m.casItem(x, e)) {         // lost CAS                CAS竞争失败
                // 上面三个条件无论哪一个满足，都证明m已经失效无用了，
                // 需要将其出队
                advanceHead(h, m);          // dequeue and retry
                continue;
            }

            // 成功匹配，依然需要将节点出队
            advanceHead(h, m);              // successfully fulfilled
            // 唤醒匹配节点，如果它被阻塞了
            LockSupport.unpark(m.waiter);
            return (x != null) ? (E)x : e;
        }
    }
}

Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    /* Same idea as TransferStack.awaitFulfill */
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = (head.next == s)
        ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS)
        : 0;
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel(e);
        Object x = s.item;
        // item被修改后返回
        // 如果put操作在此等待，item会被更新为null
        // 如果take操作再次等待，item会由null变为一个值
        if (x != e)
            return x;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        if (spins > 0) {
            --spins;
            Thread.onSpinWait();
        }
        else if (s.waiter == null)
            s.waiter = w;
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD)
            LockSupport.parkNanos(this, nanos);
    }
}
```