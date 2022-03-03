## 1 CAS

CAS(Compare and Swap)是对一种处理器指令（例如x86处理器中的cmpxchg）的称呼。

CAS需要有3个操作数：内存地址V，旧的预期值A，即将要更新的目标值B。

CAS指令执行时，当且仅当内存地址V的值与预期值A相等时，将内存地址V的值修改为B，否则就什么都不做。整个比较并替换的操作是一个原子操作。例如AtomicInteger中的getAndIncrement()方法实现

```java
public final int getAndIncrement() { 
    return unsafe.getAndAddInt(this, valueOffset, 1);   
}
//Unsafe类中的getAndAddInt方法
public final int getAndAddInt(Object o, long offset, int delta) {        
    int v;        
    do {            
        v = getIntVolatile(o, offset);        
    } while (!compareAndSwapInt(o, offset, v, v + delta));        
    return v;
}
```

例如count++实际上是一个read-modify-write操作，它可以由CAS转换为一种一般性的if-then-act操作，并由处理器保障该操作的原子性。

**CAS只能保障共享变量更新的原子性，并不保障可见性**。

CAS是非阻塞式的

## 2 java中的原子类

java中原子类相当于CAS实现的增强型volatile变量（volatile保障CAS无法保障的可见性）

原子变量类共有12个，可被分为4组，如下

|    分组    |                              类                              |
| :--------: | :----------------------------------------------------------: |
| 基础数据型 |           AtomicInteger、AtomicLong、AtomicBoolean           |
|   数组型   |  AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray   |
| 字段更新器 | AtomicIntegerFieldUpdater、AtomicIongFieldUpdater、AtomicReferenceFieldUpdater |
|   引用型   | AtomicReference、AtomicStampedReference、AtomicMarkedableReference |

字段更新器（AtomicIntegerFieldUpdater、AtomicIongFieldUpdater、AtomicReferenceFieldUpdater）这三个类相对更底层一些，可以理解为对CAS的一种封装，其他原子类都可以利用这三个类来实现。

**AtomicStampedReference可以用来解决ABA的问题**，实现方式是：为共享变量的更新引入一个修订号（时间戳）。每次更新共享变量时相应的修订号就增加1，即使变量V实际值经历了A->B->A的更新，修订号也改变了，就可以判断变量是否被其他线程修改过。

## 3 CAS存在的问题

+ **循环时间长开销很大**
  我们可以看到getAndAddInt方法执行时，如果CAS失败，会一直进行尝试。如果CAS长时间一直不成功，可能会给CPU带来很大的开销。
+ **只能保证一个共享变量的原子操作**
  当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁来保证原子性。

+ **ABA问题**

  如果内存地址V初次读取的值是A，并且在准备赋值的时候检查到它的值仍然为A，那我们就能说它的值没有被其他线程改变过了吗？

  如果在这段期间它的值曾经被改成了B，后来又被改回为A，那CAS操作就会误认为它从来没有被改变过

## 4 源码

```kotlin
inline jint   Atomic::cmpxchg  (jint   exchange_value, volatile jint*   dest, jint   compare_value) {
 int mp = os::is_MP();
 __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
          : "=a" (exchange_value)
          : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
          : "cc", "memory");
 return exchange_value;
}
```



Atomic::cmpxchg方法解析：

mp是“os::is_MP()”的返回结果，“os::is_MP()”是一个内联函数，用来判断当前系统是否为多处理器。

+ 如果当前系统是多处理器，该函数返回1。
+ 否则，返回0。

LOCK_IF_MP(mp)会根据mp的值来决定是否为cmpxchg指令添加lock前缀。

+ 如果通过mp判断当前系统是多处理器（即mp值为1），则为cmpxchg指令添加lock前缀。
+ 否则，不加lock前缀。

这是一种优化手段，认为单处理器的环境没有必要添加lock前缀，只有在多核情况下才会添加lock前缀，因为lock会导致性能下降。cmpxchg是汇编指令，作用是比较并交换操作数。

**intel手册对lock前缀的说明如下**：

1. 确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从Pentium 4，Intel Xeon及P6处理器开始，intel在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低lock前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。 
2. 禁止该指令与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

上面的第1点保证了CAS操作是一个原子操作，第2点和第3点所具有的内存屏障效果，保证了CAS同时具有volatile读和volatile写的内存语义。