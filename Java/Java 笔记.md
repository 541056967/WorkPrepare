# Java 笔记
## 1. 集合框架
集合接口分为Collections和Map，Collection分为List和Set。
### 1.1 List
List的主要实现类有ArrayList、LinkedList、Vector，都实现了Iterator接口
####  ArrayList 
* 底层实现是Object数组：`transient Object[] elementData`
* 默认长度为10，扩容后变为之前的1.5倍：`int newCapacity = oldCapacity+(oldCapacity>>1)`
* 扩容创建新的数组，`elementData = Arrays.copyOf(elementData, newCapacity)`
* 支持随机访问

#### LinkedList
* 内部维护一个双向链表实现，内部类`Node(val, next, pre)`是链表的节点，字段`first、last、size`
* 可以当做堆栈、队列或双端队列进行操作
* 不支持随机访问，但删除、修改的时间复杂度不受元素位置影响，ArrayList增加和查找很快
  
#### Vector
* 底层也是`Object[]数组`，是线程安全的，支持随机访问
* 默认长度为10，，如果指定`capacityIncrement`增长系数，每次扩容指定长度；否则每次扩容为之前的两倍
  
#### 总结
* 对于频繁查找的场景，使用ArrayList
* 对于频繁修改的场景，使用LinkedList

#### 快速失败（fail——fast）
* 在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出`Concurrent Modification Exception`
* 迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个` modCount` 变量。集合在被遍历期间如果内容发生变化，就会改变`modCount`的值。每当迭代器使用`hashNext()/next()`遍历下一个元素之前，都会检测`modCount`变量是否为`expectedmodCount`值，是的话就返回遍历；否则抛出异常，终止遍历
* 解决方案：`CopyOnWriteArrayList`来替换`ArrayList`，任何对数据的修改，`CopyOnWriterArrayList`都会copy现有的数据，再在copy的数据上修改，这样就不会影响`COWIterator`中的数据了，修改完成之后改变原有数据的引用即可。缺点是产生大量对象。
  
#### 安全失败（fail——safe）
采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。`java.util.concurrent`包下的容器都是安全失败，可以在多线程下并发使用，并发修改。有两个问题
* 需要复制集合，产生大量的无效对象，开销大
* 无法保证读取的数据是目前原始数据结构中的数据。
  
### 1.2 Set

### 1.3 Map

## 2. Java多线程
### 2.1 进程与线程
#### 并行与并发
* 并行：**单位时间内**，多个任务同时执行。
* 并发：**同一时间段**，多个任务都在执行 (单位时间内不一定同时执行)；

#### 进程与线程的区别
* 进程是程序的一次执行过程，是系统运行程序的基本单位，一个进程在执行过程中可以产生多个线程
* 与进程不同的是同类的多个线程共享进程的**堆**和**方法区**资源，但每个线程有自己的**程序计数器**、**虚拟机栈**和**本地方法栈**，所以系统在产生一个线程，或是在各个线程之间作切换工作时，负担要比进程小得多，也正因为如此，线程也被称为轻量级进程

#### 直接调用Thread的`run()`方法可以吗
* 不行，这样只是调用类中的一个方法，当做普通的一个线程，程序中只有一个**主线程**
  
#### 什么是线程安全？
* 访问共享资源或变量；依赖时序的操作；不同数据之间存在绑定关系；对方没有声明自己是线程安全的

#### 上下文切换
* 多线程编程中一般**线程的个数都大于 CPU 核心的个数**，而一个 CPU 核心在任意时刻只能被一个线程使用，为了让这些线程都能得到有效执行，CPU 采取的策略是为每个线程分配**时间片并轮转**的形式。当一个线程的时间片用完的时候就会重新处于就绪状态让给其他线程使用，这个过程就属于一次上下文切换。
* 实际上就是**任务从保存到再加载的过程就是一次上下文切换**。

#### `start()`方法源码
```java
public synchronized void start() {
    // 等于0意味着可以是线程的新建状态
    if (threadStatus != 0)
        throw new IllegalThreadStateException();
	// 将该线程加入线程组
    group.add(this);
    boolean started = false;
    try {
        start0(); // 核心， 本地方法，新建线程被。
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
        }
    }
}
```
* 当得到CPU的时间片后就会执行其中的**run()方法**。这个run()方法包含了要执行的这个线程的内容，run()方法运行结束，此线程也就终止了。
```java
@Override
public void run() {
    if (target != null) {
        target.run(); // target是：private Runnable target; Runnable接口
    }
}
// Runnable:
public abstract void run();//抽象方法
```
#### 线程的生命周期
![](https://www.pdai.tech/_images/pics/ace830df-9919-48ca-91b5-60b193f593d2.png)
* 线程创建后处于`NEW`新建状态，调用`start()`方法后开始运行，此时处于`READY`（可运行，也叫就绪）状态，当可运行的线程获得了**CPU**时间片后处于`RUNNING`状态。
* 当线程执行 `wait()`方法之后，线程进入 `WAITING`（等待）状态。进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而 `TIME_WAITING`(超时等待) 状态相当于在等待状态的基础上增加了超时限制，比如通过 `sleep（long millis）`方法或 `wait（long millis）`方法可以将 Java 线程置于 `TIMED WAITING` 状态。当超时时间到达后 Java 线程将会返回到`RUNNABLE`状态。
* 当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到 `BLOCKED`（阻塞）状态。
* 线程在执行 Runnable 的` run() `方法结束之后将会进入到 `TERMINATED`（终止） 状态。
  
#### wait/notify和sleep方法的异同
**相同点**：
* 都能够让**线程阻塞**；都可以响应**interrupt**中断，在等待过程中如果收到中断信号都可以响应并抛出**InterruptedException**异常
**不同点**
* wait 方法必须在 **synchronized** 保护的代码中使用，而 sleep 方法并没有这个要求。
* 在同步代码中**执行 sleep 方法时，并不会释放 monitor 锁，但执行 wait 方法时会主动释放 monitor 锁**
* sleep 方法中会要求必须定义一个时间，时间到期后会主动恢复，而对于没有参数的 wait 方法而言，意味着永久等待，直到被中断或被唤醒才能恢复，它并不会主动恢复。
* **wait/notify 是 Object 类的方法，而 sleep 是 Thread 类的方法**。

### 2.2 线程的创建
四种创建方式
#### 继承`Thread`
```java
public class Test extents Thread {
    public void run() {
      // 重写Thread的run方法
      System.out.println("dream");
    }
    
    public static void main(String[] args) {
        new Test().start();
    }
}
```

#### 实现Runnable接口
* 实现Runnable接口，获取实现Runnable接口的实例，作为参数，创建Thread实例，执行`Thread#start()` 启动线程
```java
public class Test implements Runnable {
 public void run() {
      // 重写Runnable的run方法
      System.out.println("dream");
    }

    public static void main(String[] args) {
       Test test = new Test();
       Thread thread = new Thread(test);
       thread.start();
    } 
}
```
#### 实现Callable接口,结合FutureTask创建线程
```java
public class Test implements Callable<Inteager> {
    public void run() {
      // 重写Callable的run方法
      System.out.println("dream");
    }

    public static void main(String[] args) {
       FutureTask<Integer> future1 = new FutureTask<>(new Test());
       Thread thread1 = new Thread(future1, "线程1");
       thread1.start();
    } 
}
```
#### 线程池创建
```java
public class Test {
    public static void main(String[] args) {
        ExecutorService threadPool = Executors.newFixedThreadPool(1);
        threadPool.submit(() -> {
            System.out.println("dream");
        });
        threadPool.shutdown();
    }
}
```
#### Runnable和Callable的区别
* `Callable`接口相比`Runnable`而言，会有结果返回，因此会由`FutrueTask`进行封装，以期待获取线程执行后的结果；最终线程的启动都是依赖`Thread.start`
  
#### 主线程等待子线程的方式
* `sleep`
* `wait`
* `CountDownLatch`
```java
  final CountDownLatch latch = new CountDownLatch(num);  //num是子线程数量
  latch.countDown()  //子线程在run方法执行结束调用
  latch.await()  //主线程等待执行
```

### 2.3 死锁
#### 死锁的条件
- **互斥条件**：该资源任意一个时刻只由一个线程占用。(同一时刻，这个碗是我的，你不能碰)
- **请求与保持条件**：一个进程因请求资源而阻塞时，对已获得的资源保持不放。（我拿着这个碗一直不放）
- **不剥夺条件**:线程已获得的资源在末使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。（我碗中的饭没吃完，你不能抢，释放权是我自己的，我想什么时候放就什么时候放）
- **循环等待条件**:若干进程之间形成一种头尾相接的循环等待资源关系。（我拿了A碗，你拿了B碗，但是我还想要你的B碗，你还想我的A碗）比喻成棒棒糖也阔以。
  
##### 解决死锁的方法
* 预防死锁
  * **资源一次性分配**：破坏请求和保持条件
  * **可剥夺资源**：当进程申请新的资源不满足时，释放已经分配的资源，破坏不可剥夺条件
  * **资源一次性分配**：系统给进程编号，按某一顺序申请资源，释放资源时则反序释放，破坏循环等待条件
* 避免死锁：银行家算法：分配资源前先评估风险，会不会在分配后导致死锁。即分配给一个进程资源的时候，该进程能否全部返还占用的资源。
* 检测死锁：建立资源分配表和进程等待表。
* 解除死锁：可以直接撤销死锁进程，或撤销代价最小的进程。

### 2.4 JMM内存模型
![volatile保证内存可见性和避免重排-WDKMZd](https://gitee.com/dreamcater/blog-img/raw/master/uPic/volatile保证内存可见性和避免重排-WDKMZd.png)
* JMM（Java Memory Model）：Java 内存模型，是 Java 虚拟机规范中所定义的一种内存模型，Java 内存模型是标准化的，屏蔽掉了底层不同计算机的区别。也就是说，JMM 是 JVM 中定义的一种并发编程的底层模型机制。
* JMM 定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。
* JMM 的规定：
  * 所有的共享变量都存储于主内存。这里所说的变量指的是实例变量和类变量，不包含局部变量，因为局部变量是线程私有的，因此不存在竞争问题。
  * 每一个线程还存在自己的工作内存，线程的工作内存，保留了被线程使用的变量的工作副本。
  * 线程对变量的所有的操作（读，取）都必须在工作内存中完成，而不能直接读写主内存中的变量。
  * 不同线程之间也不能直接访问对方工作内存中的变量，线程间变量的值的传递需要通过主内存中转来完成。

### 2.5 `volatile`关键字
#### 特性：内存可见性、禁止重排序
* 禁止重排序：不管是编译器还是JVM、CPU，都会对一些指令进行重排序，目的提高程序运行的速度，提高性能。->单例模式
* 内存可见性：当一个线程修改了某个变量的值，其他线程总是能知道这个变化。
#### 底层结构
* 禁止重排是利用内存屏障来解决的，其实最根本的还是cpu的一个**lock**指令：**它的作用是使得本CPU的Cache写入了内存，该写入动作也会引起别的CPU invalidate其Cache。所以通过这样一个空操作，可让前面volatile变量的修改对其他CPU立即可见。** 
  - 锁住内存
  - 任何读必须在写完成之后再执行
  - 使其它线程这个值的栈缓存失效

#### 不能保证原子性

### 2.6 CAS
### 2.7 ThreadLocal
### 2.8 `synchronized`关键字
#### `synchronized`和`lock`的区别
* 两者都是可**重入锁**（自己可以再次获取自己的内部锁，比如当某一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当期再次想要获取找哥哥对象的锁时还可以获取，如果锁不可重入就会造成死锁）
* `synchronized`依赖于JVM，而`lock`是JDK层面实现的，依赖于API，需要`lock()`和`unlock()`方法配合`try/finally`语句来完成。
* **ReenTrantLock 比 synchronized 增加了一些高级功能**
  * 等待可中断
  * 可实现公平锁
  * 可实现选择性通知（锁可以绑定多个条件）
  * 性能已不是选择标准
#### 修饰范围
* 实例方法：作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁
* 静态方法：作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
* 修饰代码块：指定加锁对象，对给定对象加锁，进入同步代码库前要获得指定对象的锁
#### 底层原理
- 代码块
**synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。**
- 方法
**synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。**
#### 1.6对synchronized的优化
* JDK1.6对锁的实现引入了大量的优化，如**自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等**技术来减少锁操作的开销。

### 2.9 `ReentrantLock`关键字
#### 原理
* 实现了Lock接口，同时调用内部类sync继承AQS，**state**表示获取该锁的可重入次数，在默认下，`stats==0`**表示当前锁没有被任何线程持有**。当某线程第一次获取该锁时会尝试使用CAS设置`stats=1`，并记录持有者为当前线程，第二次获取时设置为2.
#### 方法
```java
 public void lock() {
    sync.lock(); // 委托给sync了
}
```
```java
// 由fair来决定是公平锁还是非公平的
 public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
```java
    private static ReentrantLock reentrantLock = new ReentrantLock();
    reentrantLock.lock();
    finally{
        reentrantLock.unlock();
    }
```

### 2.10 CountDownLatch
#### 原理
* 状态变量`state`，表示计数器当前的值。
* 当线程调用`CountDownLatch`对象的`await`方法后，当前线程会被阻塞，直到**线程调用了`countDown`方法使得计数器的值为0**或其他线程调用了**当前线程的interrupt()方法中断了当前线程**，当前线程就会抛出`InterruptedExceptio`n异常。

### 2.11 Java的各种锁
#### 公平/非公平锁
* 公平锁指多个线程按照申请锁的顺序来获取锁
* 非公平锁指多个线程获取锁的顺序并不是按照申请的顺序
* 典型如`ReentrantLocK`

#### 可重入锁
* 可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁，典型的`synchronized`
```java
synchronized void setA() throws Exception {
  	Thread.sleep(1000);
  	setB(); // 因为获取了setA()的锁，此时调用setB()将会自动获取setB()的锁，如果不自动获取的话方法B将不会执行
}
synchronized void setB() throws Exception {
  	Thread.sleep(1000);
}
```

#### 独享锁/共享锁
- 独享锁：是指该锁一次只能被一个线程所持有。
- 共享锁：是该锁可被多个线程所持有。

#### 互斥锁/读写锁
**上面讲的独享锁/共享锁就是一种广义的说法，互斥锁/读写锁就是其具体的实现**

#### 乐观锁/悲观锁
* 乐观锁与悲观锁不是指具体的什么类型的锁，而是指看待兵法同步的角度。
* 悲观锁认为对于同一个人数据的并发操作，一定是会发生修改的，哪怕没有修改，也会认为修改。因此对于同一个数据的并发操作，悲观锁采取加锁的形式。悲观的认为，不加锁的并发操作一定会出现问题。
* 乐观锁则认为对于同一个数据的并发操作，是不会发生修改的。在更新数据的时候，会采用尝试更新，不断重新的方式更新数据。乐观的认为，不加锁的并发操作时没有事情的。
* 悲观锁适合写操作非常多的场景，乐观锁适合读操作非常多的场景，不加锁带来大量的性能提升。
* 悲观锁在Java中的使用，就是利用各种锁。乐观锁在Java中的使用，是无锁编程，常常采用的是CAS算法，典型的例子就是原子类，通过CAS自旋实现原子类操作的更新。重量级锁是悲观锁的一种，自旋锁、轻量级锁与偏向锁属于乐观锁

#### Java锁总结
* `Synchronized`是一个非公平、悲观、独享、互斥、可重入的重量级锁
* `ReentrantLock`是一个默认非公平但可实现公平的悲观、独享、互斥、可重入的重量级锁
* `ReentrantReadWriteLock`是一个默认非公平但可实现公平的、悲观、写独享、读共享、读写、可重入、重量级锁。

### 2.11 线程池
#### 线程池的特性
* **降低资源消耗**：通过重复利用已创建的线程降低创建和销毁造成的消耗
* **提高响应速度**：当任务到达时，任务可以不需要等到线程创建就立即执行
* **提高线程的可管理性**：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

#### 线程池举例
* `newFixedThreadPool`（固定线程池)
* `newSingleThreadExecutor`（单个线程的线程池)
* `newCachedThreadPool`（缓存线程的线程池)
* `newScheduledThreadPool`（带定时器的线程池）

#### 线程池参数
- `corePoolSize`：核心线程数线程数定义了最小可以同时运行的线程数量
- `maximumPoolSize`：当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数
- `keepAliveTime`：当线程数大于核心线程数时，多余的空闲线程存活的最长时间
- `TimeUnit`：时间单位
- `BlockingQueue<Runnable>`：当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，就会被存放在队列中
- `ThreadFactory`：线程工厂，用来创建线程，一般默认即可
- `RejectedExecutionHandler`：拒绝策略

#### 拒绝策略
- AbortPolicy：抛出 `RejectedExecutionException`来拒绝新任务的处理。
- CallerRunsPolicy：调用执行自己的线程运行任务。您不会任务请求。但是这种策略会降低对于新任务提交速度，影响程序的整体性能。另外，这个策略喜欢增加队列容量。如果您的应用程序可以承受此延迟并且你不能任务丢弃任何一个任务请求的话，你可以选择这个策略。（说白了，谁管理任务的，谁就负责帮忙）
- DiscardPolicy：不处理新任务，直接丢弃掉。
- DiscardOldestPolicy：此策略将丢弃最早的未处理的任务请求。

#### `shutdown`和`shutdownNow`区别
* `shutdown`等待所有任务执行完毕才推出
* `shutdownNow`立即中断所有线程，关闭线程池

#### `execute`和`submit`区别
* **可以接受的任务类型不同**：execute只能接受Runnable类型的任务；submit不管是Runnable还是Callable类型的任务都可以接受，但是Runnable返回值均为void，所以使用Future的get()获得的还是null
* **submit()有返回值，而execute()没有**
* **submit()可以进行Exception处理**

## 3.Java基础知识
### 3.1 基本类型
#### 装箱和拆箱
* 装箱：将基本类型用它们对应的**引用类型**包装起来
* 拆箱：将包装类型转换为**基本数据类型**
* 基本类型有：
  - byte（1字节）：Byte
  - short（2字节）：Short
  - int（4字节）：Integer
  - long（8字节）：Long
  - float（4字节）：Float
  - double（8字节）：Double
  - char（2字节）：Character
  - boolean（未知）：Boolean
- **在装箱的时候自动调用的是Integer的valueOf(int)方法。而在拆箱的时候自动调用的是Integer的intValue方法**

### 3.2 ==、hashcode和equals
#### **==**概念及作用
* **判断两个对象的地址是不是相等**，即判断两个对象是不是同一个对象
* 基本数据类型中，==比较的是值
* 引用数据类型中，==比较的是内存地址

#### **equals**概念及原理
* equals是顶层父类`Object`的方法之一
```java
// object默认调用的== ， 对象如果不重写，可能会发上重大事故
public boolean equals(Object obj) {
    return (this == obj); // 比较对象的地址值
}
```
```java
// Returns a hash code value for the object.
// 调用本地方法返回的就是该对象的地址值
public native int hashCode();
```
* 作用：
  * 情况1：类没有覆盖 equals() 方法。则通过 equals() 比较该类的两个对象时，等价于通过`==`比较这两个对象。
  * 情况2：类覆盖了 equals() 方法。一般，我们都覆盖 equals() 方法来比较两个对象的内容是否相等；若它们的内容相等，则返回 true (即，认为这两个对象相等)。

### 3.3 String、StringBuffer、StringBuilder的区别
#### 可变性
* `String` 类中使用 `final` 关键字修饰字符数组来保存字符串，`private　final　char[]　value，所以 `String` 对象是不可变的。（还不是为了线程安全和JVM缓存速度问题）
* `StringBuilder` 与 `StringBuffer` 都继承自 `AbstractStringBuilder` 类，在 `AbstractStringBuilder` 中也是使用字符数组保存字符串char[]value 但是没有用 `final` 关键字修饰，所以这两种对象都是可变的

#### 线程安全
* `String`中的对象是不可变的，可以理解为常亮，是线程安全的
* `AbstractStringBuilder` 是 `StringBuilder` 与 `StringBuffer` 的公共父类，定义了一些字符串的基本操作，如 `expandCapacity`、`append`、`insert`、`indexOf` 等公共方法。`StringBuffer` 对方法加了同步锁或者对调用的方法加了同步锁，所以是**线程安全**的。`StringBuilder` 并没有对方法进行加同步锁，所以是**非线程安全**的。

#### 性能
- 每次对 `String` 类型进行改变的时候，都会生成一个新的 `String` 对象，然后将指针指向新的 `String` 对象。
- `StringBuffer` 每次都会对 `StringBuffer` 对象本身进行操作，而不是生成新的对象并改变对象引用。相同情况下使用 `StringBuilder` 相比使用 `StringBuffer` 仅能获得 10%~15% 左右的性能提升，但却要冒多线程不安全的风险。

#### 总结
* 操作少量的数据，适用`String`
* 单线程操作字符串缓冲区下操作大量数据: 适用`StringBuilder`
* 多线程操作字符串缓冲区下操作大量数据: 适用`StringBuffer`

#### final关键字
* **使用场景**：变量、方法、类
  * 对于一个final变量，如果是**基本数据类型的变量，则其数值一旦在初始化之后便不能更改**；如果是引用类型的变量，则在对其初始化之后便**不能再让其指向另一个对象**。
  *  当用final修饰一个类时，表明**这个类不能被继承**。final类中的所有成员方法都会被隐式地指定为final方法。
  *  使用final方法的原因有两个。第一个原因是把方法锁定，以防任何继承类修改它的含义；第二个原因是效率。

* **好处**
  * inal的关键字**提高了性能**，JVM和java应用会**缓存final变量**；
  * final变量可以在多线程环境下保持**线程安全**；
  * 使用final的关键字提高了性能，JVM会对方法变量类进行优化；

#### `String`对象和常量池
```java
public class StringTest {
    public static void main(String[] args) {
        String str1 = "todo"; // 常量池
        String str2 = "todo"; // 从常量池找了str1
        String str3 = "to"; // 常量池
        String str4 = "do"; // 常量池
        String str5 = str3 + str4; // 内部用StringBuilder拼接了一波。 因此， 并非常量池
        String str6 = new String(str1); //  创建对象了， 那还能是常量池的引用？
    }
}
```
- 成的`class`文件中会在常量池中**保存“todo”、“to”和“do”三个String常量**。
- 变量`str1`和`str2`均保存的是常量池中“todo”的引用，所以`str1==str2`成立；
- 在执行 `str5 = str3 + str4`这句时，**JVM会先创建一个StringBuilder对象，通过StringBuilder.append()方法将str3与str4的值拼接**，然后通过`StringBuilder.toString()`返回一个堆中的`String`对象的引用，赋值给`str5`，因此`str1和str5`指向的不是同一个`String`对象，`str1 == str`5不成立；
- `String str6 = new String(str1)`一句显式创建了一个新的String对象，因此`str1 == str6`不成立便是显而易见的事了。

### 3.4 对象特性
封装、继承、多态
#### 封装
* 把一个对象私有化，同时提供一些可以被外界访问的属性的方法

#### 继承
* 使用**已存在的类**的定义作为基础建立新类的方式，新类的定义可以增加**新的数据或新的功能，也可以使用父类的功能，但不能选择性地继承父类**。通过使用继承方便地复用以前的代码。

#### 多态的定义
* 三要素：**继承、重写、父类引用只想子类对象（`Parent p = new Child()`）**

#### 多态的表现形式
* Java的方法重载，就是在类中可以创建多个方法，它们具有相同的名字，但可具有不同的参数列表、返回值类型。调用方法时通过传递的参数类型来决定具体使用哪个方法
* **Java的方法重写，是父类与子类之间的多态性，子类可继承父类中的方法，但有时子类并不想原封不动地继承父类的方法，而是想作一定的修改，这就需要采用方法的重写。重写的参数列表和返回类型均不可修改

#### 多态的底层
当程序需要运行某个类时，类加载器会将相应的class文件载入到JVM中，并在方法区建立该类的类型信息（包括方法代码、类变量、成员变量以及**方法表**）
* 方法表的结构如同字段表，依次包括了**访问标志、名称索引、描述符索引、属性表集合**几项。
* **方法表是实现动态调用的核心。为了优化对象调用方法的速度，方法区的类型信息会增加一个指针，该指针指向记录该类方法的方法表，方法表中的每一个项都是对应方法的指针**。

#### 接口和抽象类的区别
* 方法能否实现：所有方法在接口中不能有实现（JDK8中接口方法可以有默认实现），而**抽象类可以有抽象的方法。**
* 变量：接口中除了**static、final变量**，不能有其他变量，而抽象类中则不一定。
* 一个类可以实现**多个接口**，但**只能实现一个抽象类**。接口自己本身可以通过implement关键字扩展多个接口。
* 修饰符：接口方法默认修饰符是public，抽象方法可以有public、protected和default这些修饰符（抽象方法就是为了被重写所以不能使用private关键字修饰！）。
* 抽象类是对事物的抽象，接口是对行为的抽象

#### 匿名内部类传参数需要final
* 局部变量是被当成了参数传递给匿名对象的构造器（那就相当于拷贝一份，那就是浅拷贝了），这样的话，如果不管是基本类型还是引用类型，一旦这个局部变量是消失（局部变量随着方法的出栈而消失），还是数据被改变，那么此时匿名构造类是不知道的，毕竟是你浅拷贝了一份，那么如果加上final，这个数据或者引用永远都不会改变，保证数据一致性

#### 四种修饰符的限制范围
* **public**: 可以被其他所有类所访问
* **private**：只能被自己访问和修改
* **protected**：只能自身、子类及同一个包中的类访问
* **default**：同一包中的类可以访问

### 3.5 Object
```java
public final native Class<?> getClass();
public native int hashCode(); // 返回对象的哈希代码值。
public boolean equals(Object obj) 
protected native Object clone() // 创建并返回此对象的副本。
public String toString() // 返回对象的字符串表示形式。
public final native void notify(); // 唤醒正在该对象的监视器上等待的单个线程。
public final native void notifyAll(); // 唤醒正在该对象的监视器上等待的全部线程。
public final native void wait(); // 使当前线程等待，直到另一个线程调用此对象的方法或方法。
protected void finalize(); // 当垃圾回收确定不再有对对象的引用时，由对象上的垃圾回收器调用。
```

### 3.6 异常
> `Java`将程序执行中发生的不正常情况称为“异常”。
* 执行过程中所发生的的异常事件可分为两大类：
  * **Error**：Java虚拟机无法解决的严重问题，如JVM系统内部错误、资源耗尽等情况，如栈溢出和内存耗尽
  * **Exception**：其他因变成错误或偶然的外在因素导致的一般性问题，可以使用针对性的代码处理。例如空指针访问。Exception分为两大类，运行时异常（`RunTimeException`）和编译时异常(FileNotFoundException)

#### 异常体系
* `Throwable-->Error，Exception`
  * `Error--> StackOverflowError,OutOfMemoryError`
  * `Exception-->RuntimeException-->NullPointerException, ArithmeticException,ArrayIndexOutOfBoundsException, ClassCastException, NUmberFormatException`
#### 受检异常和非受检异常
* 派生于`Error和RuntimException`的所有异常都是非受检异常，所有其他的异常成为受检异常

#### try/catch/finally语句执行顺序
> finally语句总会执行，如果try{}中有return语句，先执行try（包括return表达式），再执行finally，最后返回return表达式的结果

### 3.7 值传递
- 一个方法**不能修改一个基本数据类型的参数**（即数值型或布尔型）。
- 一个方法可以改变**一个对象参数的状态**。
- 一个方法**不能让对象参数引用一个新的对象**。

### 3.8 深拷贝和浅拷贝
- **浅拷贝**：对**基本数据类型进行值传递**，对**引用数据类型进行引用传递般的拷贝**，此为浅拷贝。
- **深拷贝**：对**基本数据类型进行值传递**，对**引用数据类型，创建一个新的对象，并复制其内容，此为深拷贝**

### 3.9 反射机制
#### 反射的定义
* 在运行状态中，对于任意一个类都能获得该类所有的**属性和方法**；并且对于任意一个对象，都能调用其**任意一个方法**；这种**动态获取信息以及动态调用对象方法**的功能称为 Java 的反射机制。

#### 反射的作用
* Java的对象在运行时会出现两种类型：**编译时类型和运行时类型**，编译时类型由**声明对象时使用的类型来决定**，运行时的类型由**实际赋值给对象的类型决定** 。比如`People = = new Man()`,**编译时根本无法预知该对象和类属于哪些**类，程序只能靠运**行时信息来发现该对象和类的信息**，就要用到反射了。

#### 反射的API
* **Class类**：反射的核心类，可以获取类的属性，方法等信息
* **Field类**：`Java.lang.reflec` 包中的类，表示类的成员变量，可以用来获取和设置类之中的属性值。
* **Method类**：`Java.lang.reflec` 包中的类，表示类的方法，它可以用来获取类中的方法信息或者执行方法。
* **Constructor类**：`Java.lang.reflec` 包中的类，表示类的构造方法

#### 获取class的方式
```java
Student student = new Student(); *// 这一new 产生一个Student对象，一个Class对象。*
Class studentClass2 = Student.class; // 调用某个类的 class 属性来获取该类对应的 Class 对象
Class studentClass3 = Class.forName("com.reflect.Student") // 使用 Class 类中的 forName() 静态方法 ( 最安全 / 性能最好 )
```

- `Class.class` 的形式会使 JVM 将使用类装载器将类装入内存（前提是类还没有装入内存），**不做类的初始化工作**，返回 Class 对象。
- `Class.forName()` 的形式会装入类并做类的**静态初始化**，返回 Class 对象。
- `getClass()` 的形式会对类进行**静态初始化**、**非静态初始化**，返回引用运行时真正所指的对象（因为子对象的引用可能会赋给父对象的引用变量中）所属的类的 Class 对象。  
- 静态属性初始化是在加载类的时候初始化，而非静态属性初始化是 new 类实例对象的时候初始化。它们三种情况在生成 Class 对象的时候都会先判断内存中是否已经加载此类。
- `ClassLoader`就是遵循**双亲委派模型最终调用启动类加载器的类加载器**，实现的功能是“通过一个**类的全限定名来获取描述此类的二进制字节流**”，获取到二进制流后放到JVM中，加载的类**默认不会进行初始化**。

### 3.10 `static`关键字
#### 修饰成员变量和成员方法
* 被 static 修饰的成员属于类，不属于单个这个类的某个对象，被类中所有对象共享，可以并且建议通过类名调用。被static 声明的成员变量属于静态成员变量，静态变量存放在 Java 内存区域的方法区。调用格式：`类名.静态变量名` `类名.静态方法名()`

#### **静态代码块:** 
* 静态代码块定义在类中方法外, 静态代码块在非静态代码块之前执行(静态代码块—>非静态代码块—>构造方法)。该类不管创建多少对象，静态代码块只执行一次

#### **静态内部类（static修饰类的话只能修饰内部类）：** 
* 静态内部类与非静态内部类之间存在一个最大的区别: 非静态内部类在编译完成之后会隐含地保存着一个引用，该引用是指向创建它的外围类，但是静态内部类却没有。没有这个引用就意味着：  
  * 它的创建是不需要依赖外围类的创建。
  * 它不能使用任何外围类的非static成员变量和方法。

#### **静态导包(用来导入类中的静态资源，1.5之后的新特性):** 
* 格式为：`import static` 这两个关键字连用可以指定导入某个类中的指定静态资源，并且不需要使用类名调用类中静态成员，可以直接使用类中静态成员变量和成员方法。

### 3.11 泛型
Java SE 1.5的新特性，泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数
- **类型安全**，提供编译期间的类型检测
- **前后兼容**
- **泛化代码,代码可以更多的重复利用**
- **性能较高**，用GJ(泛型JAVA)编写的代码可以为java编译器和虚拟机带来更多的类型信息，这些信息对java程序做进一步优化提供条件

### 3.12 序列化
* 所有需要网络传输的对象都需要实现序列化接口，通过建议所有的`javaBean`都实现`Serializable`接口。
* 对象的类名、实例变量（包括基本类型，数组，对其他对象的引用）都会被序列化；方法、类变量、`transien`t实例变量都不会被序列化。
* 如果想让某个变量不被序列化，使用`transient`修饰。
* 序列化对象的引用类型成员变量，也必须是可序列化的，否则，会报错。
* 反序列化时必须有序列化对象的`class`文件。
* 当通过文件、网络来读取序列化后的对象时，必须按照实际写入的顺序读取。
* 单例类序列化，需要重写`readResolve()`方法；否则会破坏单例原则。
* 同一对象序列化多次，只有第一次序列化为二进制流，以后都只是保存序列化编号，不会重复序列化。
* 建议所有可序列化的类加上`serialVersionUID` 版本号，方便项目升级。