---
layout: post
title: "lock and condition"
description: ""
category: 
tags: []
---
# 锁和条件变量
就先从unix的Pthread库开始说起。

## Pthread库中的锁和条件变量

锁和条件变量是进行多线程应用程序开发时最常用的building block，在unix平台上开发多线程应用程序时最常用的就是unix提供的Pthread库，其中最为人熟悉两种线程同步机制就是mutex和condition variable。

mutex就是一个基本的互斥锁，当某个线程需要访问一个共享的资源时获取这个锁，处理完之后再释放锁，同一时刻只允许一个线程拥有锁，它保证了对资源的互斥访问，从而使得多个线程之间能够安全访问所共享的资源。

<!--more-->

condition variable则是另外一种线程间同步机制，它一般用于应用程序中有个共享的状态，一些操作只有在这个状态满足一定条件时才能继续执行，否则应该去wait直到状态改变。

### mutex的使用(pthread_mutex_t)
初始化和销毁
pthread_mutex_init
pthread_mutex_destory

获取锁、释放锁
pthread_mutex_lock
pthread_mutex_trylock
pthread_mutex_unlock

pthread中除了基本的互斥锁，还提供了读写锁，其使用于多读少写的应用场景。


### condition variable的使用(pthread_cond_t)
初始化和销毁
pthread_cond_init
pthread_cond_destory

等待状态
pthread_cond_wait
pthread_cond_timedwait

通知状态
pthread_cond_singal
pthread_cond_broadcast

相比于mutex，条件变量的使用就不那么trival。涉及到应用程序的共享的状态，条件变量需要一个mutex来保护多个线程对状态的并发访问。一般来说，代码如下面所示。


{% highlight c linenos %}
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

//判断状态是否满足条件：
void stateDependentMethod(){
    pthread_mutex_lock(&lock);
    while(!conditionHold()) {
        pthread_cond_wait(&lock, &cond);
    }
    /* ...perform action ...*/
    pthread_mutex_unlock(&lock);
}

//改变状态
void changeStateMethod() {
    pthread_mutex_lock(&lock);
    /* perform action that change state */
    pthread_cond_singal(); // or pthread_cond_broadcast
    pthread_mutex_unlock(&lock);
}
{% endhighlight %}

注意三个地方：

1）使用mutex保护对共享状态的访问。在这里使用了lock变量用于保护应用的共享状态，使得不会有多个线程同时改变状态。另外，由于conditionHold()判断和pthread_cond_wait()不是原子操作，如果没有加锁的话可能会有其他线程在这两个操作之间的窗口期改变状态，当前线程就会错过这次状态改变通知，加锁操作就等于关闭了这个窗口。

2) 将mutex作为参数传入pthread_cond_wait：从代码中可以看出，pthread_cond_wait传入两个参数，cond和lock。为什么需要lock呢？这就需要了解下pthread_cond_wait操作干了哪些事。pthread_cond_wait首先会释放传入的mutex，然后再阻塞当前线程，直到被其他线程调用pthread_cond_signal/broadcast唤醒。被唤醒后它将先获取mutex的锁（pthread_mutex_lock，如果锁被其他线程占用，当前线程将继续阻塞），得到这个锁后pthread_cond_wait才返回到用户代码中（即前面的while循环）。试想，如果pthread_cond_wait不去先释放锁再阻塞当前线程，别的线程怎么又机会去改变状态呢！

3）永远要把pthread_cond_wait写在while循环中。因为在多线程应用程序中，同时在某个条件变量上等待的线程可能不止一个，他们被唤醒后会对状态进行竞争，也就是说即便是被唤醒了，状态可能还是没有满足条件。

下面实现一个简单的无界的工作队列为例：(为了简单起见出队、入队放在一端进行)
{% highlight c linenos %}
struct msg {
    struct msg * next;
    /* .... */
}

struct msg *workqueue;
pthread_cond_t qready = PTHREAD_COND_INITIALIZER;//initialize a condition variable
phtread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;//initialize a mutex

void process_msg(void){
    struct msg *mp;
    for(;;){
        pthread_mutex_lock(&qlock);

        while(workqueue == NULL) {
            pthread_cond_wait(&qready, &qlock);
        }

        mp = workqueue;
        workqueue = mp->next;
        pthread_mutex_unlock(&qlock);
        /* process the message mp */
    }   
}

void enqueue_msg(struct msg *mp){
    phtread_mutex_lock(&qlock);
    mp->next = workqueue;
    workqueue = mp;
    pthread_cond_signal(&qready);
    pthread_mutex_unlock(&qlock);
}

{% endhighlight %}
在这个例子中，共享的状态是内部工作队列workqueue的状态，取任务操作能够继续的条件是workqueue不为空，同时入队任务时需要去通知等待一个在条件变量上的线程。


### 怎么做到的
对于pthread_mutex_t和pthread_cond_t类型而言，其内部都有个等待线程队列(注意区分，mutex的是锁等待队列，condition variable的是条件等待队列)。对于mutex而言，当一个线程对一个已经被占用的mutex去调用pthread_mutex_lock时，该线程则会被加入到这个mutex的等待队列中，然后交出cpu。当mutex锁的拥有者调用pthread_mutex_unlock释放锁时，取出等待队列中的一个线程，并且将其唤醒（通常是队首线程）。condition variable类似，线程调用pthread_cond_wait使得线程被加入这个条件变量内部的等待队列中。当别的线程调用pthread_cond_signal/broadcast时，取出一个或者所有等待队列中线程并唤醒。
要注意的是pthread_cond_wait干的事要多些。它的实现可以用下面的伪码描述下：
{% highlight c linenos %}
unlock mutex;
put current thread into waiting queue;
blocking current thread;
/** ....sleeping... **/
relock mutex;
{% endhighlight %}

## Java的内部锁和内部条件队列
上一节花了很多篇幅去介绍unix的提供的mutex和condition variable的使用以及其实现机制，原因之一是因为这两者是并发编程里最常用的building block。另一方面是为了更好地理解java中做法。java更是在语言层就内置了这两者，分别称为intrinsic lock和intrinsic  condition queue（我们就称之为内部锁和内部条件队列吧）。

这里先给出一个事实：java里任何对象内部都有一个锁等待队列和一个条件等待队列。

### 内部锁
了解java语言的都知道，java提供了synchronized关键字，用于保护临界区代码（可能是访问一个共享的资源）。常见：
{% highlight java linenos %}
// balabala ...
synchronized (obj) {
    // balabala ...
}
// balabala ...
{% endhighlight %}
多个线程将以互斥、串行的方式执行synchronized块内代码。其实synchronized的语义和mutex的语义完全一样，当线程进入synchronized块时先获取obj对象的锁，获取到了继续执行，获取不到则等待直到拥有者释放obj的锁（类比pthread_mutex_lock）。线程退出synchronized块时释放obj的锁，从而唤醒正在等待obj对象锁的线程(类比pthread_mutex_unlock)。

synchronized块的括号内的obj可以是任何类型的对象，原因就是前面提到的，java的任何对象内部都有一个锁等待队列，也就是任何对象都能充当一个mutex。

### 内部条件队列
java里所有类都有wait,notify,notifyAll三个方法，继承自Object类。这三个方法构成了java的内部条件队列API，也就是说java任何对象也都能充当一个条件变量。不同于Pthread中mutex和condition variable是分开且独立的两个变量（分别是pthread_mutex_t和pthread_cond_t类型），java将条件变量和互斥锁放在同一个对象里面，因此使得java使用条件变量变得简单，典型的写法如下：
{% highlight java linenos %}
//加锁
synchronized (lock) {
    while (!conditionPredicate()) {
        lock.wait();
    }
    //条件满足后执行动作
    /* ...perform action... */
} //释放锁
{% endhighlight %}

{% highlight java linenos %}
synchronized (lock) {
    /* ...perform action that make the precondition become true ...*/
    notify(); /* or notifyAll() */ 
}
{% endhighlight %}

下面以实现一个线程安全的有界BlockingQueue作为使用java内部条件队列机制的例子，代码如下：

{% highlight java linenos %}
public class BlockingBoundedQueue<V> extends BaseBoundQueue<V> {
    // condition predicate : not-full (!isFull())
    // condition predicate : not-empty (!isEmpty())
    
    public BoundedBuffer(int size) { super(size);}
    
    //block until not-full
    public synchronized void put(V v) throws InterruptedException {
        while(isFull()) {
            wait();
        }
        doPut(v);
        notifyAll();
    }
    
    //block until not-empty
    public synchronized V take() throws InterruptedException {
        while(isEmpty()) {
            wait();
        }
        V v = doTake();
        notifyAll();
        return v;
    }
}
{% endhighlight %}
BlockingBoundedQueue继承自BaseBoundQueue，BaseBoundQueue代码这里就不给出了，基本上就是一个简单的数组实现的queue，支持对元素的doPut、doTake以及isFull、isEmpty方法。BlockingBoundedQueue扩展了它的功能，增加了Blocking行为，使得当queue为空时，take操作阻塞直到非空；当queue满时，put操作阻塞直到非满。

在BlockingBoundedQueue实现中，所有线程所共享的状态是底层queue的状态。无论是锁对象还是条件变量对象都是用的this对象（即当前BlockingBoundedQueue的实例），然而存在两个预置条件（非空和非满），它们使用的是同一个内部条件队列(this对象的内部条件队列)。换句话说，在当前BlockingBoundedQueue的实例内部的条件队列中等待着两类线程，一类等待着非空条件，一类等待着非满条件。因此无论是put还是take操作，都需要调用notifyAll()来通知所有线程状态发生了改变，而不能使用notify()。这就好比两个人共用一个电话，当来电话了只去通知一个人肯定是不行的。

那么什么情况下才可以去使用notify()呢？必须满足以下所有条件：

- 统一的等待线程：所有线程等待同一个预置条件，即条件队列中的所有线程等待的预置条件是相同的，而且，所有线程的执行逻辑是一致。

- 一进一出：一次状态改变的通知只允许一个线程收到并处理。

显然，BlockingBoundedQueue的例子满足一进一出的条件，但是不满足统一的等待线程条件。

看上去，所有能使用notify的地方都可以使用notifyAll，反之则不然，那么干脆所有地方都直接使用notifyAll算了，这样好么？当然不好，**`notifyAll是有代价的！`**

试想在一个**`一次状态改变只允许一个线程收到并处理`**的场景下使用notifyAll，比如内部条件队列中有十个等待线程，它们都被这个notifyAll所唤醒，都会去竞争获取内部对象锁，但最终只能有一个线程能获得锁，其他又都回去睡觉了。这意味着一次notifyAll将产生很多次无意义的线程上下文切换和锁竞争，毫无疑问将影响性能。

怎么破？对于使用内部条件队列的BlockingBoundedQueue实现而言，这个问题是无解。好在java cocurrent包提供了所谓的显式锁和显式条件队列。


## java显式锁和显式条件队列
自java5.0开始，Doug Lea为JDK贡献了大名鼎鼎的concurrent包，这里面不仅提供了很多新的并发编程的常用组件，另外还提供了一些性能和功能上有很大改善的java原有基础组件的替代品。比如ReentrantLock之于java内部锁，Condition之于内部条件队列。

### 显示锁
最常用的显式锁是ReentrantLock，它实现了Lock接口，下面是Lock接口的定义，从中也可以看到它比内部锁功能强的地方。
{% highlight java linenos %}
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
{% endhighlight %}

ReentrantLock实现了Lock接口，它提供了和synchronized一直的互斥功能和内存可见性保证。synchronized代表的内部锁最大的问题线程一旦阻塞，没法去中断阻塞，意味着如果一直不能获取到锁，线程将一直阻塞下去；另外，内部锁的获取和释放代码总是位于一个方法内部的某个代码块内，使得其没法实现那些非块状结构的锁使用情况（比如在一个方法中获取锁，在另外一个方法中释放锁）。ReentrantLock则通过提供多种获取锁方法和一个释放锁方法避免了上述问题。

ReentrantLock的标准使用方式如下：
{% highlight java linenos %}
Lock lock = new ReentrantLock();
/* ...balabala... */
lock.lock();
try {
    //临界区代码
} finally {
    lock.unlock();
}
{% endhighlight %}

- tryLock()和tryLock(long timeout, TimeUnit unit)，ReentrantLock提供了非阻塞版本的尝试获取锁方法tryLock()和带时间阻塞版本的tryLock(long timeout, TimeUnit unit)，通过它们可以实现一些避免死锁的策略（比如随机退避）。

- lockInterruptibly()提供了可中断的获取锁方法，使得因调用这个方法的线程可以响应来自其他线程的interrupt()请求。

- Non-block-structured Locking，ReentrantLock使得使用者可以在一个方法中lock()，另一个方法中unlock()，大大提高了灵活性。

- 性能上的提升，在java6之前，ReentrantLock相对于java的内部锁提供了非常大的性能提升。java6改良了管理内部锁的算法，使得两者性能基本一致了。因此，现在不能再说显式锁比内部锁性能好了!

#### Fairness（公平性）
ReentrantLock除了上述几点不同于内部锁外，还提供了关于对配置锁公平性的功能。首先介绍下什么是锁的公平性，试想一个场景：目前已经有多个线程在一个锁的等待队列里等待，此时拥有这个锁的线程释放了该锁（lock.unlock()），与此同时，一个新的线程(比如是T)过来准备获取这个锁(lock.lock)，那么关于T线程要不要去lock的锁等待队列中排队就是锁公平性的体现。如果lock是非公平的锁，它将允许后来者T直接抢占这个锁，而无须排队；如果lock是公平的，它将使得T老老实实的去排队，而唤醒队首线程，让它获得锁。

绝对公平一定意味着好么？答案是否。从上面的描述中可知，公平锁使得正处于running状态的线程T切换为sleeping状态（suspending），使得队首处于sleeping状态的线程切换为ready状态(resuming)，这一切都需要操作系统的介入，都需要进行线程上下文切换，公平性的代价是很大的。实际应用中，除非公平性是算法正确性的前提，一般没必要使用公平锁。ReentrantLock默认情况以及内部锁都是非绝对公平的（但保证统计上的公平）。当然可以给ReentrantLock传个boolean参数，让其变为公平锁。

#### 关于内部锁和显式锁的选择
到目前为止，ReentrantLock各方面都不输于内部锁，也的确是这样的，那是否意味是我们干脆直接使用ReentrantLock替代内部锁算了，答案是否。原因有两点：

- 第一，内部锁写法简单。虽然显式锁写法也很简单，但是如果哪个缺心眼的程序员忘了将unlock操作包装在finally块中，那就等于在程序中安装了一颗定时炸弹，会伤及无辜的！

- 第二，前面也说了，java6后内部锁的性能有很大的改善，已经和ReentrantLock不相上下，后续还有继续提升的可能。另外，由于内部锁是语言层面内置的，一些聪明JVM可以做些不必要的锁消除和锁粒度粗化优化，这一点基于concurrent库的ReentrantLock就没法做到了。

因此，结论是当你想使用ReentrantLock提供的灵活功能时使用ReentrantLock，否则synchronized就行了。另外concurrent包中还有个有用的ReadWriteLock接口。它的定义如下：
{% highlight java linenos %}
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
{% endhighlight %}
它非常适用于那些多读少写的应用场景，使用也很简单，这里就不展开了。


### 显示条件队列
还记得Lock接口有个newCondition()方法吧，它返回一个与当前锁对象关联的Condition对象，即一个显示条件队列对象。Condition是个接口，定义如下：
{% highlight java linenos %}
public interface Condition {
    void await() throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    void awaitUninterruptibly();
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
{% endhighlight %}

内部锁的wait、notify、notifyAll和Condition的await、signal、signalAll功能对应。Condition还提供了带时间版本的await和可中断的await。

前面说过，每个java对象内部都一个锁等待队列和一个条件等待队列，也就说内部锁和内部条件队列是一一对应的，这也是前面BlockingBoundedQueue不得已使用notifyAll的原因。显示条件队列则不同，对于一个显式锁对象，你可以去调用newCondition()创建任意多个Condition对象，也就说现在可以关联多个Condition对象到同一个Lock对象上了！下面将利用这点实现一个性能上优于BlockingBoundedQueue的ConditionBoundedQueue，代码如下：

{% highlight java linenos %}
public class ConditionBoundedQueue<T> {
    protected final Lock lock = new ReentrantLock();
    
    // CONDITION PREDICATE: notFull (count < items.length)
    private final Condition notFull  = lock.newCondition();
    
    // CONDITION PREDICATE: notEmpty (count > 0)
    private final Condition notEmpty  = lock.newCondition();
    
    @GuardedBy("lock")
    private final T[] items = (T[]) new Object[BUFFER_SIZE];
    @GuardedBy("lock")
    private int tail, head, count;
    
    // BLOCKS-UNTIL: notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[tail] = x;
            if (++tail == items.length)
                tail = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    
    
    // BLOCKS-UNTIL: notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            T x = items[head];
            items[head] = null;
            if (++head == items.length)
                head = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
{% endhighlight %}


# concurrent包中其他常用的building block
前面介绍了concurrent包中两个比较底层的building block(Lock & Condition)，当你需要开发一些并发组件、容器时考虑使用这些显式锁&显式条件队列替换内部锁&内部条件队列。这一节中将介绍一些应用开发中常用的high level的building block。

## 并发集合

### ConcurrentHashMap
java5.0中增加了ConcurrentHashMap，作为对之前线程安全的Map实现(Hashtable、SynchronizedMap)的替代。ConcurrentHashMap相对于Hashtable等在多线程环境下有着非常好并发性能(基本可以达到一个数量级的提升)。

ConcurrentHashMap是通过一个常见的并发优化技巧——lock Striping来做到性能提升的。其基本思想很简单，就是减少锁的保护范围。Hashtable实现中每个读写操作都需要锁住整个哈希表，我们知道哈希表有哈希桶的概念，不同元素的hashcode相同的话则落在同一个哈希桶中，同一个哈希桶中元素可以通过一个链表组织起来。那么不同桶中元素的读写是独立的，显然Hashtable这个任何操作都先锁住这个哈希表的做法是有优化空间的。 ConcurrentHashMap的做法就是分配16个锁，每个锁负责保护（1/16*总哈希桶数）个哈希桶，如果元素在哈希表中分布均匀的话，那么并发度大约就提升16倍。

下面以一个简单hashmap实现StripedMap演示下lock striping技巧。
{% highlight java linenos %}
public class StripedMap {
    // Synchronization policy: buckets[n] guarded by locks[n%N_LOCKS]
    private static final int N_LOCKS = 16;
    private final Node[] buckets;
    private final Object[] locks;
    private static class Node { ... }
    
    public StripedMap(int numBuckets) {
        buckets = new Node[numBuckets];
        locks = new Object[N_LOCKS];
        for (int i = 0; i < N_LOCKS; i++){
            locks[i] = new Object();
        }
    }
    
    private final int hash(Object key) {
        return Math.abs(key.hashCode() % buckets.length);
    }
    
    public Object get(Object key) {
        int hash = hash(key);
        synchronized (locks[hash % N_LOCKS]) {
            for (Node m = buckets[hash]; m != null; m = m.next)
                if (m.key.equals(key))
                    return m.value;
        }
        return null;
    }

    public void clear() {
        for (int i = 0; i < buckets.length; i++) {
            synchronized (locks[i % N_LOCKS]) {
                buckets[i] = null;
            } 
        }
    }
    ...
}
{% endhighlight %}

### copy-on-write collections
CopyOnWriteArrayList是对synchronized List(比如Vector)的替代，CopyOnWriteArraySet则是对synchronized Set的替代。copy-on-write集合的性能也很大程度上优于他们的被替代品。copy-on-write底层是一个数组实现，它的并发性能源自于多线程对它的读操作压根就不用加锁；当有线程执行写操作时，先直接分配一个大小和原先底层数组大小一致的数组，copy下所有元素，再对某个元素赋值，最后替换掉集合对象的底层数组，这就是所谓的copy-on-write。
以add方法为例，看下CopyOnWriteArrayList的实现：

{% highlight java linenos %}
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements); //将底层数组引用赋值为新分配的newElements
        return true;
    } finally {
        lock.unlock();
    }
}
{% endhighlight %}

这里ReentrantLock的使用时为了保证setArray(newElements)对所有线程正确可见性。

显然，CopyOnWrite集合不适用于那些有着非常多元素的大集合，因为一旦有写操作就要去拷贝整个集合，这代价也不小。CopyOnWrite集合非常适合于那些读、遍历集合操作要远多于写操作的场景，比如事件通知的注册listeners集合，此时delivering一个事件远比向register注册一个listener要来得频繁。


## BlockingQueue
多线程应用程序中最常见的编程模型之一就是生产者-消费者模型，任何一本操作系统的书籍中都会反复提到这个。生产者-消费者在大量框架、容器中都有着很广泛的应用，比如java的Executor Framework中、jetty实现中、web爬虫中等等处处都是它的影子。顺便安利一个我写的[抵用券发券服务](http://wiki.sankuai.com/pages/viewpage.action?pageId=461933580)，核心的结构就是一个生产者-消费者模型。BlockingQueue就是一个用于实现生产者-消费者模型的常用组件。它底层是一个Queue，一般来说，它表现出如下行为：当Queue为空时，从中取任务操作将被阻塞，直到Queue非空；当Queue为满时，往Queue中塞任务操作将被阻塞，直到Queue非满（如果是无界BlockingQueue的话则不存在Queue满的情况）。

java提供了好几个BlockingQueue的实现，LinkedBlockingQueue和ArrayBlockingQueue都是FIFO类型的BlockingQueue，一个底层是链表，一个则是数组。 

ArrayBlockingQueue的实现就类似于前面的ConditionBoundedQueue。
{% highlight java linenos %}
/** Main lock guarding all access */
final ReentrantLock lock;

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;
{% endhighlight %}

取任务和塞任务操作都需要先获取这一个lock，即生产者和消费者之间是会存在竞争的。另外ArrayBlockingQueue大小的计数器(count)采用基本类型int实现，使得size()方法实现时也需要去获取这个全局的lock，然而这点使我很不解（明明count可以用原子变量实现的。。）

LinkedBlockingQueue的实现就很符合直觉。
{% highlight java linenos %}
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();

/** Current number of elements */
private final AtomicInteger count = new AtomicInteger();
{% endhighlight %}

可以看出，用了两个lock，takeLock和putLock，由于take和put分别在链表的两端进行，这两个操作间是可以并发的，也就说队列的生产者和消费者之间不会产生锁竞争。看到这个之后，我将抵用券发券服务的核心数据结构——内存中的任务队列，以LinkedBlockingQueue替换原来的ArrayBlockingQueue，单机QPS直接提升了2K，因此我的结论是：在ArrayBlockingQueue和LinkedBlockingQueue中优先选择后者，毕竟他俩对外提供的功能一致，区别仅在于底层实现（有人可能要说，数组相对于链表而言有很好的空间局部性，呵呵呵，naive!）。

除了FIFO queue之外，还有个PriorityBlockingQueue实现，它底层是个优先级队列，也是个很常见的东西（基于最小堆实现），不多说了。

## 各种Synchronizers
前面介绍了concurrent包中的几个集合以及BlockingQueue，相对而言BlockingQueue即充当了集合的角色，也可以去来根据状态来协调多个线程的执行。这节中介绍几个单纯用于协调多线程执行的类，称为Synchronizer，从这个定义来看，锁和条件队列也是Synchronizer。

### latch
latch的意思是门闩，它的行为也像一个门闩，一般来说，一个latch有个终止状态，在达到状态前，所有使用这个latch的线程将被阻塞。concurrent包提供一个CountDownLatch类，使用也很简单，它暴露两个方法await和countDown，await使得调用者在这个latch上等待，countDown则用于改变latch的内部状态。CountDownLatch的构造方法需传入一个整数，表示需要countDown几次才能到达这个latch的终止状态。使用例子如下：

{% highlight java linenos %}
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);
        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    /* preprocess */
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException ignored) { }
                }
            };
            t.start(); 
        }
        
        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end-start;
    }
}
{% endhighlight %}
在上面的例子中，timeTasks方法中创建了多个线程并发的执行某个task，但是需要确保所有线程在执行task.run()之前都全部都预处理完成，另外，调用线程还需要统计下所有线程都处理完时所花费的时间。因此需要两个latch用来为调用线程和工作线程协调。

### FutureTask
FutureTask就更常见了，Executor framework用它来表示一个异步任务，通常来说FutureTask内部有个有返回结果计算任务，它实现了Runnable和Future接口，分别实现了run()和get()方法。也就是说，可以去直接调用一个FutureTask的run方法在调用线程中执行计算任务，此外还可以把FutureTask作为参数传给一个Thread对象，然后调用Thread对象的start方法，在这个线程中执行计算任务。get()方法行为依赖于内部计算任务的状态，如果计算任务完成了get将返回计算结果，否则将阻塞调用线程。
构造时传入一个Callable接口的实现类对象作为要执行的计算任务。

例子就不举了，因此遍地都是。

### Semaphores
信号量也是操作系统书籍里的常客，concurrent包中的信号量的行为和一般理解的信号量行为一致，即构造时传入一个整形的permit（允许多少个线程能够accquire到这个信号量），提供两个方法accquire和release。每次accquire会使得permit减一，release使得permit加1，permit为0时，accquire使得调用线程阻塞，直到其他线程release。如果permit为1，那么信号量的行为就是一个互斥锁。

信号量非常适合用来实现资源池（比如数据库连接池），通常对于固定大小的资源池而言，可以初始化一个permit为资源池大小的信号量，每当有线程请求资源时去调用accquire，释放资源时调用release。

例子同样不举。

### Barriers
Barrier有点类似于Latch，最大的区别在于在latch上await的线程需要别的线程去CountDown来改变latch的状态，然而Barrier没有提供额外方法让线程去改变内部状态。await方法本身就会改变Barrier的内部状态。怎么说呢，以CyclicBarrier类为例，它的构造方法需传入一个整形参数，表示需要汇聚到的线程个数parties，当线程调用await时，如果此时在这个barrier上等待的线程数小于parties，那么await将阻塞调用线程，一旦在barrier上await的线程上达到了parties，那么barrier将立即唤醒所有await的线程，让他们继续执行下去。

另外，latch是一次性的，一旦到达了它的终止状态，latch就没用了（再有别的线程来await时也不会阻塞）。CyclicBarrier不同，它是可循环利用的，一旦barrier上等待线程攒满了parties，barrier同时唤醒所有线程，同时自身的状态重置，依旧可以继续使用。

CyclicBarrier最常用于物理模拟计算中，比如常见的粒子系统就非常适合使用多个线程来并发处理，比如采用最简单的欧拉积分方法来对粒子系统进行运动模拟，每次迭代计算前需要保证所有线程已经完成对整个粒子系统的上一次迭代计算，此时一个CyclicBarrier就可以很好地协调多个线程。

## AQS framework
OK，上一节中介绍了多个Sychronzier，他们行为不同，干的事情也不一样，但神奇的是他们大多数底层居然是基于同一个东西实现的，这个东西就是AbstractQueuedSynchronizer，即Doug Lea贡献的AQS framework，这里不得不佩服Doug Lea的抽象能力。

这些Sychronizer虽然表现的行为不同，其实抽象地来看他们还是有个共同点，即他们内部都有个状态，且大多可以抽象出两类操作——accquire和release，accquire是一个依赖状态的操作，并且可能会阻塞调用线程。对于Lock和Semaphores而言，accquire很直白（获取锁、获取一个permit）；对于CountDownLatch而言，accquire意味着等待latch到达它的终止状态；对于FutureTask而言，accquire意味着等待计算任务执行完成。release则是个非阻塞操作，它将改变内部状态，并且可能会唤醒因accquire而阻塞的线程。

AQS统一以一个int型的整数来表示状态，并用volatile关键字修饰，保证了多线程下，state的内存可见性。
{% highlight java linenos %}
/**
 * The synchronization state.
 */
private volatile int state;
{% endhighlight %}

accquire和release的伪代码如下：
{% highlight java linenos %}
boolean acquire() throws InterruptedException {
    while (state does not permit acquire) {
        if (blocking acquisition requested) {
            enqueue current thread if not already queued
            block current thread
        } else
            return failure
    }
    possibly update synchronization state
    dequeue thread if it was queued
    return success
}
void release() {
    update synchronization state
    if (new state may permit a blocked thread to acquire)
        unblock one or more queued threads
}
{% endhighlight %}

看起来是不是很眼熟，没错，和之前介绍条件队列使用时给出的伪码很类似，AQS内部的核心数据结构也是个双向链表实现的队列(CLH lock queue)。区别在于条件队列中enqueue和dequeue操作是在获取到一个互斥锁的前提下进行的，而这里是没有锁的，那么它是怎么保证在多线程下的链表的线程安全的？答案是它以一种lock free的方式来访问底层的链表，基于CPU提供CAS指令对链表的HEAD和TAIL指针进行写操作。具体细节还有很多，这里不展开了，有兴趣的话可自行查阅源代码。

# more ..
这篇wiki绝大部分篇幅都在介绍锁以及相关的Synchronizers，但是锁不是银弹，不能包治百病，尤其是在一些性能要求高的应用场景下，锁反而不是一个好的选择，要知道锁是有开销的，是需要操作系统介入并进行线程的上下文切换的。concurrent包中也在将之前的一些基于锁实现的组件以non-blocking算法重写，比如Semaphore和ConcurrentLinkedQueue。大多的non-blocking算法都使用到CPU的CAS指令（乐观锁）。在java5.0之前，没有直接提供CAS相关的支持，如果想使用CPU的CAS指令，只能去写native的代码。5.0开始，增加了一些封装CAS指令的底层类(AtomicXXX)，JVM也开始支持将使用这些类的代码编译成给定平台上的CAS指令。

然而设计正确的non-blocking算法不是那么容易的，JVM和JDK中库的作者们也一直在这个方向上努力着，毕竟non-blocking算法带来的性能提升和其他优点是非常诱人的。

关于一些常用并发数据结构的non-blocking算法实现在《java concurrency in pratice》这本书中有个很好的描述。业界也涌现出了很多优秀的non-blocking实现的开源产品，比如大名鼎鼎的[Disruptor](http://lmax-exchange.github.io/disruptor/)，一款in-memory的并发队列。这篇[文章](http://martinfowler.com/articles/lmax.html)中详细介绍了基于Disrupt实现的LMAX（a new retail financial trading platform），号称能达到单线程每秒处理6M订单，这是要上天的啊！这是Disruptor的作者的[博客](http://mechanitis.blogspot.hk/2011/06/dissecting-disruptor-how-do-i-read-from.html)，其中通俗易懂地介绍了Disruptor怎么做的，为啥能有这么高的性能。

# Reference
本文内容主要出自已故技术作家Stevens的《APUE》、作者阵容均是JVM、JDK作者的《Java Concurreny In Pratice》、Doug Lea大神的concurrent包源码、以及自己的实践经验。

最后。以上内容纯手动码成，如果看完后有收获，请赏个赞，（羞愧脸