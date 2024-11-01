## Java中的锁

### 基础认识

#### 一、Lock接口

##### 1.介绍一下Lock接口

Lock接口是一种用于线程同步的高级机制，它位于Java并发包中。主要的实现类是ReentrantLock，它提供了一种比synchronized更加灵活、更加精细的并发控制机制。它主要的一些方法包括lock、tryLock、lockInterruptibly等。

lock方法用于获取锁，如果其他线程已经占有了锁，那么当前线程就会阻塞等待，直到其他线程释放锁之后，它会尝试再去获取锁。tryLock方法允许当前线程在非阻塞的时候继续尝试去获取锁。tryLock方法尝试获取锁，如果获取成功就立即返回true，如果锁已经被其他线程占用，则返回false，不会阻塞线程。tryLock还可以指定时间，在给定时间内尝试获取锁，如果超时仍未获取锁，则返回false。这个方法适用于不希望无限等待的场景。

lockInterruptibly方法用于获取锁，但允许在等待过程中被中断，这样可以避免无限等待的情况。

Lock接口相对于synchronized的一个优点是可中断锁，通过lockInterruptibly方法允许线程在等待的过程中被中断。tryLock方法允许线程在不阻塞的情况下获取锁，适合在可能产生锁竞争的场景中使用。而超时获取锁的方法则是在指定时间内尝试获取锁，如果获取成功则返回true，如果获取不到则自动返回false。

Lock需要我们手动操作，synchronized是JVM隐式释放锁。所以，在使用Lock的时候，unlock要放在finally代码块中，而lock不要放在try代码块中。

**为什么lock.lock()建议放在try的外部？**

因为lock方法放在内部，正常获取锁，finally就会正常释放锁；但一旦异常，并没有真正拿到锁，此时执行finally中的释放锁，就会发生IllegalMonitorStateException异常。

如果lock方法放在外部，正常获取锁，finally就会正常释放锁；但一旦异常，就不会再次进入到finally中再次引发其他异常。

```java
lock.lock();
try {
    // 代码逻辑
} finally {
    lock.unlock(); // 确保锁被释放
}
```

##### 2.怎么使用Lock接口

```java
lock.lock();
try {
    // 代码逻辑
} finally {
    lock.unlock(); // 确保锁被释放
}
```

##### 3.Lock接口提供的API有哪些？

| **API方法**                                                  | **描述**                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `void lock()`                                                | 获取锁。当前线程将会阻塞直到获取锁，当成功获取锁后才返回。   |
| `void lockInterruptibly() throws InterruptedException`       | 可中断的获取锁。与`lock()`不同，该方法允许在等待锁的过程中响应中断，抛出`InterruptedException`。 |
| `boolean tryLock()`                                          | 尝试非阻塞获取锁。立即返回，如果获取成功则返回`true`，否则返回`false`。 |
| `boolean tryLock(long time, TimeUnit unit) throws InterruptedException` | 尝试在指定的超时时间内获取锁。在以下三种情况返回：1. 成功获取锁，返回`true`；2. 等待过程中被中断；3. 超时后未获得锁，返回`false`。 |
| `void unlock()`                                              | 释放锁。释放后其他等待该锁的线程可以尝试获取锁。             |
| `Condition newCondition()`                                   | 获取一个与当前锁绑定的`Condition`条件对象，允许实现更灵活的等待和通知机制。当前线程必须持有锁后才能调用该对象的`await()`方法。 |

##### 4.介绍一下Lock和AQS之间的关系

在Java并发编程中，Lock接口提供了线程同步的机制。相比于synchronized，它的控制更加灵活和精细。Lock接口提供了如lock、tryLock、unlock、lockInterruptibly和condition等方法，让我们能够更精细地控制线程同步。

AQS（AbstractQueuedSynchronizer）是Java并发包中的一个抽象类，它为实现像Lock这样的锁机制以及CountDownLatch同步器提供了底层的基础框架。AQS维护了一个同步队列（FIFO队列）。当一个线程尝试获取锁失败时，该线程会被加入到同步队列中等待。

正是因为AQS有这样的底层机制，所以它支持线程的独占模式和共享模式。独占模式是保证同一时刻，只有一个线程可以持有锁，例如ReentrantLock，当一个线程尝试获取锁时，如果失败，它会加入到等待队列中，直到锁被释放。共享模式则允许多个线程同时占有锁，例如在ReentrantReadWriteLock中的读锁。AQS在这种情况下允许多个线程并发访问。

AQS内部通过一个int类型的state变量来判断锁是否被占用。如果这个值等于0，则表示未被占用；如果大于0，则表示已经被占用。

AQS的实现依赖于CAS操作来保证状态的线程安全变更。

##### 5.ReentrantLock和AQS的结合

ReentrantLock和AQS的关系。ReentrantLock分为公平锁和非公平锁，它实现了lock接口，并继承了AQS的特性，通过AQS来实现其独占逻辑。

公平锁是指所有尝试获取锁的线程，如果失败，会进入FIFO的同步队列中。这样，每个线程获取锁的优先级是公平的。

非公平锁则允许线程插队获取锁，从而提高吞吐量。AQS会在CAS操作成功后直接获取锁，不必进入队列。

##### 6.Lock和synchronized有什么区别？如何选择？

==synchronized 锁只能同时被一个线程拥有，但是 Lock 锁没有这个限制：例如读写锁中的读锁时可以被多个线程同时拥有的。==

synchronized和Lock是Java并发编程中实现线程同步的两个重要机制。synchronized是Java内部的一个关键字，由JVM实现。它的一些特点包括：

1. **不可中断**：一个线程一旦进入synchronized的方法或者代码块，就不可中断，直到锁被释放或者发生异常。
2. **自动管理**：由于synchronized由JVM管理，所以锁的强占和释放都是自动的，即使发生异常，锁的释放也是自动的。
3. **持续优化**：synchronized在Java的多个版本中持续升级，包括偏向锁、轻量级锁和重量级锁等一系列优化。

synchronized适合在简单的线程同步场景中使用。

接着是Lock接口，它来自Java并发包中的一个工具。其底层基于AQS框架实现，内部有一个同步队列和一个状态变量来维护。Lock接口的特点是相比于synchronized，它可以提供更加精细化、更加灵活的使用场景。例如：

- 手动管理锁的释放和抢占
- 提供了lockInterruptibly，在线程阻塞时可以进行中断。
- 可以尝试获取锁。
- 通过提供Condition条件变量来实现更加精细化的线程同步控制。

Lock接口的具体实现类包括ReentrantLock，它基于AQS实现独占锁模式，支持公平锁和非公平锁两种模式。还有ReadWriteLock，它适合在高并发、大量竞争的线程环境中使用。

总而言之，synchronized适合在简单的线程环境中使用，保证同一共享资源在指定时间内只能被一个线程使用。而Lock更适合在高并发、大量竞争的线程环境中使用。

业务选择中，尽可能选择JUC并发包中的机制使用，优先推荐工具（如ConcurrentHashMap、Atomic类、CountDownLatch、Semaphore、BlockingQueue等）；其次，才是synchronized，因为它基于JVM提供自动的抢占和释放锁；最后，才是lock锁，比如：中断、尝试、超时功能等，才使用lock。



#### 二、可重入锁ReentrantLock

##### 1.如何实现可重入锁？

可重入锁是指一个线程在尝试获取锁之后，可以再次获取同一把锁而不会被阻塞。它主要是为了解决递归调用获取锁以及多次获取同一把锁时可能发生的死锁问题。在Java中，典型的可重入锁有两种：synchronized和ReentrantLock。

synchronized是Java内置的关键字，由JVM实现隐式锁。锁的获取和释放都是由JVM控制的，通常在较为简单的业务场景中使用synchronized。synchronized还有一个锁升级的优化过程，从偏向锁到轻量级锁再到重量级锁，这是synchronized的一个特点。

```java
public class ReentrantExample {
    public synchronized void methodA() {
        System.out.println("Method A");
        methodB(); // 递归调用
    }

    public synchronized void methodB() {
        System.out.println("Method B");
    }

    public static void main(String[] args) {
        ReentrantExample example = new ReentrantExample();
        example.methodA(); // methodA 调用 methodB，使用同一个锁
    }
}
```

ReentrantLock实现了Lock接口，底层基于AQS（AbstractQueuedSynchronizer）技术框架。使用ReentrantLock时，需要手动创建和释放锁，它可以提供更加精细的线程控制和管理。例如，中断锁（lockInterruptibly方法）和超时等待（tryLock方法）等精细控制。同时，ReentrantLock提供了公平锁和非公平锁机制，以应对特定场景的需求。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private final Lock lock = new ReentrantLock();

    public void methodA() {
        lock.lock();
        try {
            System.out.println("Method A");
            methodB(); // 可以递归调用
        } finally {
            lock.unlock();
        }
    }

    public void methodB() {
        lock.lock();
        try {
            System.out.println("Method B");
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        ReentrantLockExample example = new ReentrantLockExample();
        example.methodA();
    }
}
```

可重入锁的实现原理是：内部维护了一个计数器，当一个线程获取锁一次，计数器递增；相反，当一个线程释放锁一次，计数器递减。

总的来说，在一般场景下，我们可以选择Java并发包中的工具，例如ReentrantLock。如果业务场景特别简单，只是为了避免线程抢占共享资源，可以使用synchronized，因为它可以自动释放锁。最后，我们不太推荐直接使用Lock接口进行线程同步。

##### 2.公平和非公平锁的实现？

###### （1）类的继承关系

![image-20241030225749161](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/image-20241030225749161.png?AI_make_money=VX_AI19858122061)

ReentrantLock提供了公平锁和非公平锁两种机制。ReentrantLock继承自Object，实现了Lock接口。它的底层基于AQS（AbstractQueuedSynchronizer）这个抽象类，AQS提供了两种模式：独占锁模式和共享锁模式。ReentrantLock基于独占锁模式实现了公平锁和非公平锁。

###### （2）类的内部类

ReentrantLock中有两个内部类，分别实现了公平锁和非公平锁。公平锁的内部类是FairSync，它继承自AQS，实现了公平策略，确保每个线程按照队列顺序获取锁。

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

非公平锁的内部类是NonFairSync，同样继承自AQS，但它不受队列限制，允许线程插队直接获取锁。这两个内部类实现了对应的公平锁和非公平锁机制。

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

###### （3）类的构造函数

对于ReentrantLock这个类，它有两种构造函数：一种是默认构造函数，使用非公平锁；另一种是带参数的构造函数，如果参数为true，则使用公平锁，如果为false，则使用非公平锁。这两种构造函数创建了对应的公平锁和非公平锁。

总结：

公平锁和非公平锁的实现机制有所不同。公平锁在FairSync类中实现，tryAcquire方法会让线程先检查队列中是否有其他线程，如果有，则排队。非公平锁在NonFairSync类中实现，tryAcquire方法会忽略等待队列，直接获取锁。

公平锁和非公平锁各有优劣。公平锁可以避免线程饥饿现象，但性能相对较低。非公平锁允许线程随时抢占锁，提高了吞吐量和性能，但可能导致部分线程一直无法执行任务，出现饥饿现象。

#### 三、读写锁ReentrantReadWriteLock

##### 1.什么是读写锁？

ReentrantReadWriteLock，它是一个读写锁。允许多个线程同时获取读锁，只允许单个线程独占写锁。通过读和写的互斥性来提高多线程在并发场景中的性能。ReentrantReadWriteLock的内部分为ReadLock和WriteLock。它允许多个线程同时获取读锁，只要没有写锁，读锁之间是没有互斥性的。它主要应用在读多写少的场景中，而写锁只允许一个线程独占写锁，当一个线程独占写锁时，其他线程的读写操作都会被阻塞。

##### 2.读写锁有什么特性？

ReentrantReadWriteLock有三个特性。第一个特性是它支持可重入性，即当多个线程获取读锁后，它们还可以再次获取读锁。当一个线程获取写锁后，它还可以再次获取同样的写锁，并且还能获取读锁。第二个特性是它可以实现锁的降级，即获取写锁后再获取读锁，然后释放写锁，就可以降级为读锁。第三个特性是它支持公平锁和非公平锁的获取方式。

##### 3.读写锁的接口与示例？

| API 方法                  | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `int getReadLockCount()`  | 返回当前读锁被获取的次数。注意该次数不一定等于读锁的线程数，例如同一个线程可以连续获取（重入）N次读锁，实际占用读锁的线程数为1，但该方法返回N。 |
| `int getReadHoldCount()`  | 返回当前线程获取读锁的次数。此方法在Java 6中加入到`ReentrantReadWriteLock`中，使用`ThreadLocal`保存当前线程获取读锁的次数。 |
| `boolean isWriteLocked()` | 判断写锁是否被获取。                                         |
| `int getWriteHoldCount()` | 返回当前写锁被获取的次数。由于写锁也是可重入的，因此同一线程可以连续获取多次写锁，此方法返回写锁的获取次数。 |

##### 4.读写锁的实现原理？

读写锁是多线程环境中，允许多个线程可以共享读锁，但只允许单个线程独占写锁；这种将读写分离的互斥性，保证了并发的性能。在Java并发包JUC中，ReentrantReadWriteLock是一种典型的读写锁。

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockExample {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private int sharedData = 0;

    public void write(int data) {
        rwLock.writeLock().lock();
        try {
            sharedData = data;
            System.out.println("Write: " + data);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public int read() {
        rwLock.readLock().lock();
        try {
            System.out.println("Read: " + sharedData);
            return sharedData;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public static void main(String[] args) {
        ReadWriteLockExample example = new ReadWriteLockExample();

        // 示例：读写线程
        new Thread(() -> example.write(42)).start();
        new Thread(() -> System.out.println(example.read())).start();
    }
}
```

ReentrantReadWriteLock继承Object并且实现了Lock接口，底层基于AQS框架，内部分为ReadLock和WriteLock，实现了读写锁。

- ReadLock，允许多个线程可以共享读锁，依赖于AQS的共享模式。

- WriteLock，只允许单个线程可以独占写锁，依赖于AQS的独占模式。

ReentrantReadWriteLock实现了读写锁的基础上，也支持了锁的降级。在获取写锁的基础上，继续获取读锁，同时释放写锁，最终保持在读锁状态，通过这种方式实现了锁降级，保证了不会直接释放写锁而导致数据不一致的情况。

除此之外，ReentrantReadWriteLock还在内部通过一个state的整数变量维护了读写锁的状态，低16位表示写锁的写入次数，高16位表示读锁的持有次数，通过这种方式，可以保证：

- 获取读锁时，检查写状态是否为0，如果是则获取写锁。
- 获取写锁时，检查读写状态是否都为0，如果是则成功获取写锁。

上述是从读写锁以及内部原理的角度来描述：ReentrantReadWriteLock；其实，ReentrantReadWriteLock也可以提供公平锁和非公平锁，依旧是基于AQS。

- 公平锁：所有请求的线程都会在AQS内部的同步队列中进行排队，依次获取锁。
- 非公平锁：线程可以忽视AQS内部的同步队列，直接抢锁；性能更高，但是：容易出现饥饿现象。

总体而言，读写锁特别适合读多写少的场景，例如：配置文件、缓存系统、数据监控等。

#### 四、LockSupport工具

相比sleep提供了更精细的线程阻塞和唤醒功能。

#### 五、Condition工具

Condition接口是Java中提供的一种用于线程通信的机制，一般和Lock(ReentrantLock)配合使用；效果类似：synchronized和wait、notify、notifyAll的搭配。但是，Condition提供了更加精细的等待、唤醒的控制，同时允许创建多个等待队列实现更加复杂的线程通信场景。

![image-20241031112423918](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/image-20241031112423918.png?AI_make_money=VX_AI19858122061)

- 一个Condition包含一个等待队列，Condition拥有首节点 （firstWaiter） 和尾节点（lastWaiter）；
- 一个同步器（AQS）拥有一个同步队列和多个等待队列，因为condition对象可以创建多个。（对比来看，Object监视器模型上，一个对象拥有一个同步队列和一个等待队列）

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionExample {
    private final Lock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
    private final int[] buffer = new int[10];
    private int count, putIndex, takeIndex;

    public void put(int value) throws InterruptedException {
        lock.lock();
        try {
            while (count == buffer.length) {
                notFull.await(); // 等待缓冲区有空位
            }
            buffer[putIndex] = value;
            putIndex = (putIndex + 1) % buffer.length;
            count++;
            notEmpty.signal(); // 通知有数据可取
        } finally {
            lock.unlock();
        }
    }

    public int take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // 等待缓冲区有数据
            }
            int value = buffer[takeIndex];
            takeIndex = (takeIndex + 1) % buffer.length;
            count--;
            notFull.signal(); // 通知有空位可写
            return value;
        } finally {
            lock.unlock();
        }
    }
}
```

| 方法                                      | 描述                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| `void await()`                            | 使当前线程进入等待状态，直到被其他线程通知或中断。与`Object`的`wait()`方法类似。 |
| `void awaitUninterruptibly()`             | 使当前线程进入等待状态，但不会在等待过程中响应中断。         |
| `boolean await(long time, TimeUnit unit)` | 使当前线程进入等待状态，直到被通知、中断或达到指定的等待时间。 |
| `long awaitNanos(long nanosTimeout)`      | 使当前线程进入等待状态，直到被通知、中断或达到指定的纳秒级等待时间。返回剩余的等待时间，0或负数表示已超时。 |
| `void signal()`                           | 唤醒一个等待中的线程，该线程从`await()`状态恢复。与`Object`的`notify()`方法类似。 |
| `void signalAll()`                        | 唤醒所有等待中的线程，与`Object`的`notifyAll()`方法类似。    |

### 面试题

#### 一、Lock

##### 1.Lock接口和synchronized同步对比，它有什么优势？

##### 2.怎么理解Lock与AQS的关系？

#### 二、AQS

> 复杂的文字部分就不用看了，直接听：https://ls8sck0zrg.feishu.cn/wiki/PDHVwDi05iXelzkPUqUc1yMmnPe

##### 1.什么是AQS？

##### 2.AQS怎么实现同步管理？底层数据结构是怎样的？

##### 3.AQS有哪些核心方法？

#### 三、ReentrantLock

##### 1.什么是可重入，什么是可重入锁？

##### 2.公平锁和非公平锁有什么区别？

##### 3.为什么非公平锁比公平锁性能更好？

##### 4.ReentrantLock是如何实现公平锁？非公平锁的？

##### 5.Lock和synchronized的区别是什么？

##### 6. synchronized和ReentrantLock有什么区别呢

https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Java%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e9%9d%a2%e8%af%95%e7%b2%be%e8%ae%b2/15%20%20synchronized%e5%92%8cReentrantLock%e6%9c%89%e4%bb%80%e4%b9%88%e5%8c%ba%e5%88%ab%e5%91%a2%ef%bc%9f-%e6%9e%81%e5%ae%a2%e6%97%b6%e9%97%b4.md

#### 四、ReentrantReadWriteLock

##### 1.ReentrantReadWriteLock是什么？

##### 2.共享锁和独占锁有什么区别？

##### 3.线程持有读锁还能获取写锁吗？

##### 4.什么是锁的升级？ReentrantReadWriteLock为什么不支持锁的升级？

##### 5.ReentrantReadWriteLock底层读写状态是如何设计的？



### 查漏补缺&融会贯通（重点）

https://tech.meituan.com/2018/11/15/java-lock.html

![img](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/7f749fc8.png?AI_make_money=VX_AI19858122061)

#### 一、乐观锁和悲观锁

![image-20241030100437290](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/image-20241030100437290.png?AI_make_money=VX_AI19858122061)

1.乐观锁和悲观锁是什么？适用场景是什么？

2.CAS无锁算法是什么？

3.ABA问题及解决方案？

4.CAS三大问题的解决方案？

#### 二、自旋锁和适应性自旋锁

![image-20241030100412075](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/image-20241030100412075.png?AI_make_money=VX_AI19858122061)

#### 三、无锁、偏向锁、轻量级锁、重量级锁

![image-20241030100345647](https://typora-xubang.oss-cn-hangzhou.aliyuncs.com/2024_xubang/image-20241030100345647.png?AI_make_money=VX_AI19858122061)

#### 四、公平锁和非公平锁

#### 五、可重入锁和非可重入锁

#### 六、独享锁和共享锁





