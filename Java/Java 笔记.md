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

