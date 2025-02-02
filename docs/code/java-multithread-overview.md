---
title: "Java 显式锁 Lock 与条件队列"
date: 2018-11-19T15:58:17+08:00
subtitle: ""
tags: ["Java"]
categories: ["code"]
toc: true
 
draft: false
original: false
addwechat: false
notoc: true
slug: "java-multithread-overview"
---


在 Java 5.0 之前，在协调对共享对象的访问时可以使用的机制只有 `synchronized` 内置锁和 `volatile` 关键字。

Java 5.0 增加了一种新的机制：`Lock`  显式锁，当内置锁 `synchronized` 不适用时，它就可以作为一种新的选择。

回顾一下内置锁 synchronized 的使用：

```java
// synchronized关键字用法示例
public synchronized void add(int t){// 同步方法
    this.v += t;
}

public static synchronized void sub(int t){// 同步静态方法
    value -= t;
}
public int decrementAndGet(){
    synchronized(obj){// 同步代码块
        return --v;
    }
}
```

内置锁不需要显式的获取和释放，任何一个对象都能作为一把内置锁。

*	当 synchronized 作用于普通方法时，锁对象是 this 。
*	当 synchronized 作用于静态方法时，锁对象是当前类的 Class 对象。
*	当 synchronized 作用于代码块时，锁对象是 synchronized(obj) 中的这个 obj 对象。

---


<!--more-->

## 显式锁 Lock


`ReentrantLock` 实现了 `Lock` 接口，它定义了一组抽象的加锁方法。

```java
    public interface Lock{
	    // 阻塞直到获得锁或者中断
        void lock();
        // 阻塞直到获得锁或者中断异常
        void lockInterruptibly() throws InterruptedException;
        // 只有锁可用时才获得，否则直接返回
        boolean tryLock();
        // 只有锁在指定时间内可用才获得，否则直接返回，中断时抛异常
        boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
        // 释放锁
        void unlock();
        // 返回一个绑定在这个锁上的条件
        Condition newCondition();
    }	
```

与内置锁不同，Lock 提供的锁有如下特性：

*	无条件的
*	可轮询的
*	可定时的
*	可中断的

并且加锁和解锁的方法都是显式的，而内置锁就隐式调用的。

Lock 有这些新的特性，对比的都是内置锁的痛点，通过 synchronized 来获得锁，如果得不到，就会一直阻塞，既没有超时机制，也不能通过中断来取消。


Lock 接口的标准使用形式如下：

```java
	    Lock lock = new ReentrantLock();
        lock.lock();
        try {
            // 更新对象状态
            // 捕获异常，并在必要时恢复不变性条件
        } finally {
            lock.unlock();
        }
```

Lock 的使用必须在 finally 块中释放锁，否则，如果在被保护的代码中抛出了异常，那么这个锁永远都无法释放。

当使用加锁时，还必须考虑在 try 块中抛出异常的情况，如果可能使对象处于某种不一致的状态，那么就需要更多的 try-catch 或者 try-finally 代码块。

Lock 接口提供的不同方法，实际上就对应了不同的特性。

### Lock 锁的可轮询特性

Lock 的 `tryLock()` 方法对应着可轮询的特性。

`tryLock` 方法顾名思义就是去尝试获得锁，并且具有返回值，如果锁可用，则获取锁，并立即返回 true ，若不可用，立即返回 false 。

利用此特性可以实现可轮询地获取锁：

```java
    public boolean lockUse(final Runnable task){

        Lock lock1 = new ReentrantLock();
        Lock lock2 = new ReentrantLock();
        // 超时时间
        long stopTime = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(100);
        while (true){
            if (lock1.tryLock()){
                try {
                    if (lock2.tryLock()){
                        try {
                            task.run();
                            return true;
                        }finally {
                            lock2.unlock();
                        }
                    }
                }finally {
                    lock1.unlock();
                }
            }
            // 在
            if (System.nanoTime() < stopTime){
                return false;
            }
        }
    }
```


### Lock 锁的可定时特性

Lock 的 `tryLock(long time, TimeUnit unit)` 方法对应着可轮询的特性。

带参数的 `tryLock` 方法会在一定时间范围内去尝试获得锁，如果锁可用，则获取锁，并立即返回 true ，若不可用，并且超出了等待时间就会返回 false 。 

```java
public boolean lockUse2(int time,final Runnable task) throws InterruptedException {
        Lock lock1 = new ReentrantLock();
        if (!lock1.tryLock(time,TimeUnit.MICROSECONDS)){
            return false;
        }
        try {
            task.run();
            return true;
        }finally {
            lock1.unlock();
        }
    }
```


### Lock 锁的可中断特性

Lock 的 `lockInterruptibly()` 方法对应着可中断的特性。

当通过 `lockInterruptibly` 方法去获得锁时，如果线程正在等待的过程中，那么 `Thread.interrupt()` 方法可以中断等待状态。

而 synchronized 内置锁，当线程处于等待某个锁的状态时是无法被中断的，只有一直等待下去。

```java
    public boolean lockUse3(int time,final Runnable task) throws InterruptedException {

        Lock lock1 = new ReentrantLock();

        lock1.lockInterruptibly();
        try {
            task.run();
            return true;
        }finally {
            lock1.unlock();
        }
    }
```

## 可重入锁 ReentrantLock 

可重入锁 ReentrantLock 类实现了 Lock 接口，并且具有可重入的特性，另外，synchronized 的内置锁也具有可重入的特性。

可重入特性指的是同一线程的外层函数获得锁之后，内层递归函数仍然能够获取该锁，不受影响。

可重入锁能够避免死锁，比如：

```java
public class ReentrantLockTest implements Runnable{

    ReentrantLock lock = new ReentrantLock();

    public void get(){
        lock.lock();
        System.out.println("get do something");
        set();
        lock.unlock();
    }

    public void set(){
        lock.lock();
        System.out.println("set do something");
        lock.unlock();
    }

    @Override
    public void run() {
        get();
    }

    public static void main(String[] args) {
        ReentrantLockTest test = new ReentrantLockTest();
        new Thread(test).start();
        new Thread(test).start();
        new Thread(test).start();
    }
}
```

在 get 方法中调用 set 方法，这两个方法会相同锁的 `lock` 方法。如果 ReentrantLock 不具备可重入特性，那么 set 方法中的 lock 调用就会阻塞，导致 get 方法不能执行到释放锁的步骤，从而死锁了。


## 公平锁与非公平锁

ReentrantLock 根据构造参数的不同可以实现公平锁与非公平锁，构造函数传入 true 就是公平锁，false 就是非公平锁。

默认情况下，ReentrantLock 是非公平的锁。在公平的锁上，线程将按照它们发出请求的顺序来获得锁，但在非公平的锁上，则允许 `插队` :

>  当一个线程请求非公平的锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中所有的等待线程并获得这个锁。

非公平的 ReentrantLock 并不提倡 `插队` 行为，但无法防止某个线程在合适的时候进行插队。

在公平的锁中，如果有另一个线程持有这个锁或者有其他线程在队列中等待这个锁，那么新发出请求的线程将被放入队列中。在非公平的锁中，只有当锁被某个线程持有时，新发出请求的线程才会被放入队列中。

### synchronized 与 ReentrantLock 之间的选择

ReentrantLock 在加锁和内存上提供的语义与内置锁相同，此外它还提供了一些其他功能，包括定时的锁等待、可中断的锁等待、公平性等，在性能上，ReentrantLock 似乎优于内置锁。

但与显示锁相比，内置锁仍然具有很大的优势。内置锁更被人所熟悉，并且简洁紧凑，JVM 也会对内置进行优化。

在一些内置锁无法满足需求的情况下，ReentrantLock 可以作为一种高级工具来使用。或者当需要使用 ReentrantLock 的一些高级功能时，否则，还是应该优先使用 synchronized 。

## 读写锁 ReadWriteLock

ReentrantLock 实现了一种标准的互斥锁：

> 每次最多只有一个线程能持有 ReentrantLock 。

对于维护数据的完整性来说，互斥通常是一种过于强硬的加锁规则，因此也就不必要地限制了并发性。

互斥是一种保守的加锁策略，虽然可以避免 `写 / 写` 冲突和 `写 / 读` 冲突，但同样也避免了 `读 / 读` 冲突。

在许多情况下，数据结构上的操作都是 `读操作` ，虽然它们也是可变的并且在某些情况下被修改，但其中大多数访问操作都是读操作。如果此时能够放宽加锁需求，允许多个执行读操作的线程同时访问数据结构，那么将提升程序的性能。

只要每个线程都能确保读取到最新的数据，并且在读取数据时不会有其他线程修改数据，那么就不会发生问题，在这种情况下就可以使用 读 / 写锁：`ReadWriteLock` 。

> 一个资源可以被多个读操作访问，或者被一个写操作访问，但两者不能同时进行。

```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

在读 / 写锁实现的加锁策略中，允许多个读操作同时进行，但每次只允许一个写操作。

与 Lock 接口一样，ReadWriteLock 也有多种不同的实现方式。

`ReentrantReadWriteLock` 类实现了 ReadWriteLock 接口，它为读取锁和写入锁都提供了可重入的加锁语义。

与 ReentrantLock 类似，ReentrantReadWriteLock 在构造时也可以选择是一个非公平的锁（默认）还是一个公平的锁。

在公平的锁中，等待时间最长的线程将优先获得锁，如果这个锁由读线程持有，而另一个线程请求写入锁，那么其他读线程都不能获取读取锁，直到写线程使用完并且释放了写入锁。

在非公平的锁中，线程获得访问许可的顺序是不确定的。写线程降级为读线程是可以的，但从读线程升级为写线程则是不可以的，会导致死锁。

### 读写锁 与 独占锁之间的选择

读 / 写锁是一种性能优化措施，在一些特定的情况下能够实现更高的并发性。

在实际情况中，对于在多处理器系统上被频繁读取的数据结构，读 / 写锁能够提高性能。而在其他情况下，读 / 写锁的性能比独占锁的性能要略差一点，这是因为它们的复杂性更高。

在读 / 写锁 和 独占锁之间做选择时，最好先对程序进行分析，如果读 / 写并没有提高性能，那么使用独占锁也可以。 

下面是使用 `ReentrantReadWriteLock` 来包装一 Map ，让它能够在多个线程之间被安全地共享，并且仍能够避免 读 / 写 或 写 / 写冲突。


```java
public class ReadWriteMap<K,V> {
    private final Map<K,V> map;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock r = lock.readLock();
    private final Lock w = lock.writeLock();

    public ReadWriteMap(Map<K,V> map){
        this.map = map;
    }

    public V put(K key,V value){
        w.lock();
        try {
            return map.put(key,value);
        } finally {
          w.unlock();
        }
    }

    public V get(Object key){
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
}
```


### 小结

与内置锁相比，显示的 Lock 提供了一些拓展功能，在处理锁的不可用性方面有着更高的灵活性，并且对队列有着更好的控制。但 ReentrantLock 不能完全替代 synchronized ，只有在 synchronized 无法满足需求时，才应该使用它。

读 / 写锁允许多个线程并发地访问被保护的对象，当访问以读取操作为主的数据结构时，它能够提高程序的可伸缩性。

---

## 条件谓词与条件队列

条件谓词和条件队列是平时接触比较少的内容，这里也一并记录下。

### 状态依赖性的管理

依赖状态的操作可以一直阻塞直到可以继续执行，这比使它们先失败再实现起来更为方便且不宜出错。

内置的条件队列可以使线程一直阻塞，直到对象进入某个进程可以继续执行的状态，并且当被阻塞的线程可以执行时再唤醒它们。


`条件队列` 来源于：它使得一组线程（称之为等待线程集合）能够通过某种方式来等待特定的条件变成真。传统队列的元素是一个个数据，而与之不同的是，条件队列中的元素是一个个正在等待相关条件的线程。

正如每个 Java 对象都可以作为一个锁，每个对象同样可以作为一个条件队列，并且 Object 中的 wait、notify 和 notifyAll 方法就构成了内部条件队列的 API 。

对象的内置锁与其内部条件队列是相互关联的，要调用对象 X 中条件队列的任何一个方法，必须持有对象 X 上的锁。这是因为 “等待由状态构成的条件” 与  “维护状态一致性” 这两种机制必须被紧密绑定在一起：只有能对状态进行检查时，才能在某个条件上等待，并且只有能修改状态时，才能从条件等待中释放另一个线程。


条件队列使构建高效以及高可响应性的状态依赖类变得更容易，但同时也很容易被不正确地使用。

> 在条件等待中存在一种重要的三元关系，包括加锁、wait 方法和一个条件谓词。在条件谓词中包含多个状态变量，而状态变量由一个锁来保护，因此在测试条件谓词之前必须先持有这个锁。锁对象和条件队列对象（即调用 wait 和 notify 等方法所在的对象）必须是同一个对象。


> 每一次 wait 调用都会隐式地与特定的条件谓词关联起来。当调用某个特定条件谓词的 wait 时，调用者必须已经持有与条件队列相关的锁，并且这个锁必须保护着构成条件谓词的状态变量。


避免过早唤醒，或者唤醒之后，条件谓词并不为真，使用 while 循环在调用 wait 方法。

线程在条件谓词不为真的情况下也可以反复地醒来，因此必须在一个循环中调用 wait ，并在每次迭代中都检测条件谓词。

### 通知

每当在等待一个条件时，一定要确保在条件谓词变为真时通过某种方式发出通知。

优先使用 notifyAll 而不是 notify 。

只有同时满足以下两个条件时，才能用单一的 notify 而不是 notifyAll 。

*	所有等待线程的类型都相同。只有一个条件谓词与条件队列相关，并且每个线程在从 wait 返回后将执行相同的操作。
*	单进单出。在条件变量上的每次通知，最多只能唤醒一个线程来执行。


## 显式的 Condition 对象

内置条件队列存在一些缺陷。每个内置锁都只能有一个相关联的条件队列，因而存在多个线程可能在同一个条件队列上等待不同的条件谓词，并且在最常见的加锁模式下公开条件队列对象。

这些因素都使得无法满足在使用 notifyAll 时所有等待线程为同一类型的需求。

如果想编写一个带有多个条件谓词的并发对象，或者想获得除了条件队列可见性之外的更多控制权，就可以使用显示的 Lock 和 Condition 而不是内置锁和条件队列。

一个 Condition 和一个 Lock 关联在一起，就像一个条件队列和一个内置锁相关联一样。要创建一个 Condition，可以在相关联的 Lock 上调用 Lock.newCondition 方法。正如 Lock 比内置加锁提供了更为丰富的功能，Condition 同样比内置条件队列提供了更为丰富的功能：

> 在每个锁上可存在多个等待、条件等待可以是可中断的或不可中断的、基于时限的等待，以及公平的或非公平的队列操作。

与内置条件队列不同的是，对于每个 Lock ，可以有任意数量的 Condition 对象。Condition 对象继承了相关的 Lock 对象的公平性，对于公平的锁，线程会依照 FIFO 顺序从 Condition.await 中释放。


> 在 Condition 对象中，与 wait、notify 和 notifyAll 方法对应的分别是 await、signal 和 signalAll 。但是，Condition 对 Object 进行了拓展，因而它也包含 wait 和 notify 方法。一定要确保使用正确的版本 --- await 和 signal 。


```java
public class ConditionBoundBuffer<T> {
    
    protected final Lock lock = new ReentrantLock();
    // 条件谓词： notFull (count < items.length)
    private final Condition notFull= lock.newCondition();
    // 条件谓词： notEmpty (count > 0)
    private final Condition notEmpty = lock.newCondition();
    private final T[] items = (T[]) new Object[10];
    private int tail,head,count;
    
    // 阻塞并直到：notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
                items[tail] = x;
                if (++tail == items.length){
                    tail=0;
                }
                ++count;
                notEmpty.signal();
            
        }finally {
            lock.unlock();
        }
    }
    
    // 阻塞并直到：notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0){
                notEmpty.await();
            }
            T x = items[head];
            items[head] = null;
            if (++head == items.length){
                head=0;
            }
            --count;
            notFull.signal();
            return x;
        }finally {
            lock.lock();
        }
    }
}
```
## 参考

1. 《Java 并发编程实战》
2. https://www.cnblogs.com/CarpenterLee/p/7896361.html
3. https://blog.csdn.net/justloveyou_/article/details/54972105