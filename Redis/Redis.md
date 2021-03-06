# Redis
## 1. Redis模型
### Redis为什么这么快？
* 纯内存操作
* 单线程操作，避免了**频繁的上下文切换**
* 合理高效的数据结构
* 采用了**非阻塞I/O多路复用**机制
> Redis服务器是一个事件驱动程序，分为**文件事件和时间事件**
### Redis基于Reactor模式开发了自己的网络事件处理器，称为文件处理器
* 文件事件处理器使用**I/O多路复用程序来同时监听多个套接字**，并根据套接字目前执行的任务来为套接字**关联不同的事件处理器**
* 当被监听的套接字准备好执行连接**应答、读取、写入、关闭**等操作时，与操作相对应的文件事件就会产生，这是**文件事件处理器就会调用套接字之前关联好的事件处理器来处理**
* 一堆套接字请求被一个叫做i/o多路复用的程序监听，通过文件事件分派器一个一个和事件处理器绑定在一起去处理

### 模型包括：
> 阻塞I/O模型，非阻塞I/O模型，I/O复用模型，信号驱动I/O模型，异步I/O模型

## 2. Redis数据结构
### String
* redis最基本的key-value数据类型；
* String是二进制安全的，value不仅可以是String，可以包含任何数据，比如图片和序列化的对象，最大能存储512MB
![string和hash](https://hunter-image.oss-cn-beijing.aliyuncs.com/redis/%E5%AF%B9%E8%B1%A1%E7%AF%87/%E5%93%88%E5%B8%8C/Redis-Hash.png)

### Hash
* hash是一个键值（key=>value)对集合
* hash是一个String类型的field和value的映射表，特别适合存储对象
* 相当于value是一个哈希表

### List
> 字符串列表，按照插入顺序排序,相当于value是List

### Set
* Set 是 string 类型的无序集合。集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。
* Set可以自动排重

### Zest（Sorted Set）
* 和set相比，sorted set增加了一个**权重参数score**，使得集合中的元素能够按score进行**有序**排列。

## 3. Redis的持久化模型
> **RDB和AOF**
### RDB
> **RDB**是一种**快照存储持久化**方式，具体是将`redis`某一时刻的内存数据保存到硬盘的文件当中，默认文件名为`dump.rdb`。在Redis服务器启动时，会重新加载`dump.rdb`文件的数据到内存中恢复数据
#### 优点：
* RDB会生成多个数据文件，**每个数据文件都代表了某一时刻中redis的数据**，这种多个数据文件的方式，非常适合**做冷备**
* RDB对redis对外提供读写服务的时候，影响非常小，因为redis主进程只需要fork一个子线程出来，让子进程对磁盘io化来进行rdb持久化
* **RDB在恢复大数据集时的速度比AOF的恢复速度要快**
#### 缺点：
* **如果redis要故障时要尽可能少的丢失数据，RDB没有AOF好**，例如1:00进行的快照，在1:10又要进行快照的时候宕机了，这个时候就会丢失10分钟的数据。
* RDB每次fork出子进程来执行RDB快照生成文件时，如果文件特别大，可能会导致客户端提供服务暂停数毫秒或者几秒

### AOF
> 把所有的**对Redis服务器进行修改的命令都存到一个文件里（命令的集合）**。使用AOF进行持久化，每一个写命令都通过write函数追加到`appendonly.aof`
#### 优点
* **AOF可以更好地保护数据不丢失，一般AOF每隔一秒进行一次fsync**，通过后台的一个线程去执行一次fsync操作，如果redis进程挂掉，最多丢失1秒的数据。
* AOF以appen-only的模式写入，所以没有任何磁盘寻址的开销，写入性能非常高
* AOF日志文件的命令通过非常可读的方式进行记录，非常适合做灾难性的误删除紧急恢复

#### 缺点：
* 对于同一份文件，AOF文件比RDB数据快照要大
* AOF开启后支持写的QPS会比RDB支持的写的QPS低，因为AOF一般会配置成每秒fsync操作，每秒的fsync操作还是很高
* **数据恢复比较慢，不适合做冷备**。

### 如何选择
> 用AOF来保证数据不丢失，作为恢复数据的第一选择；用RDB来做不同程度的冷备，在AOF文件都丢失或损坏不可用的时候，使用RDB进行快速恢复。

## 4. 内存淘汰机制
### Redis的过期时间（定期删除+惰性删除）
* **定期删除：**redis默认每隔100ms就随机抽取一些设置了过期时间的key，检查其是否过期，如果过期就删除。随机抽取的原因，如果每隔100ms遍历所有设置过期时间的key，会给CPU带来很大的负载
* **惰性删除**：定期删除可能会导致很多过期key没有被删除掉。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存里，除非你的系统去查一下那个key，才会被redis给删除掉。这就是所谓的惰性删除

### 内存淘汰机制
> redia内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。redis提供6种数据淘汰策略
* volatile-lu：从已经设置过期时间的数据集中挑选最少使用的数据淘汰
* volatile-ttl：从已经设置过期时间的数据集中挑选将要过期的数据淘汰
* volatile-random：从已经设置过期时间的数据集中任意选择数据淘汰
* alllkeys-lru：从数据集中挑选最少使用的数据淘汰
* alllkeys-random：从数据集中任意选择数据淘汰
* no-enviction：禁止驱逐数据

## 5. 缓存穿透和缓存雪崩
### 缓存穿透
> 一般是黑客故意去请求缓存中不存在的数据，导致所有的请求都落到数据库上，造成数据短时间承受大量请求而崩掉
* 在接口做校验
* 存null值（缓存击穿加锁）
* 布隆过滤器拦截

### 缓存雪崩
> 缓存同一时间大规模失效，所以后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉
* 使用redis高可用架构：使用redis集群来保证redis服务不会挂掉
* 缓存时间不一致：给缓存的失效时间加上一个随机值，避免集体失效
* 限流降级策略：有一定的备案，比如个性推荐服务不可用了，换成热点数据推荐服务

## 6. Redis分布式锁
### RedLock算法流程

