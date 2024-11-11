
`Semaphore` 是 Java 并发包 (`java.util.concurrent`) 中的重要工具，主要用于控制多线程对共享资源的并发访问量。它可以设置“许可证”（permit）的数量，并允许指定数量的线程同时访问某一资源，适合限流、资源池等场景。下面从源码设计、底层原理、应用场景、以及与其它 JUC 工具的对比来详细剖析 Semaphore。


### 一、Semaphore 的基本原理


`Semaphore` 本质上是一种计数信号量，内部维护一个许可计数，每个线程在进入时需要申请一个许可（acquire），完成后释放该许可（release）。当许可计数为零时，其他线程会阻塞，直到有线程释放许可。


#### 1\. 计数和许可


* **许可数**：Semaphore 在初始化时设置许可数，通常表示可以同时访问资源的线程数量。
* **计数增减**：调用 `acquire()` 时减少许可数，调用 `release()` 时增加许可数。
* **公平性**：Semaphore 可以设置为“公平”模式，保证线程按 FIFO 顺序获得许可。默认是“非公平”模式，性能更高，但线程获取许可的顺序不保证。


### 二、底层实现与源码解析


Semaphore 基于 `AbstractQueuedSynchronizer` (AQS) 实现，其实现方式和 `CountDownLatch` 类似，但使用了 AQS 的共享模式，并对许可计数进行精确管理。


#### 1\. AQS 的共享模式


Semaphore 使用 AQS 的共享模式（Shared），其中 AQS 的 `state` 表示剩余许可数。`acquireShared()` 方法用于申请许可，`releaseShared()` 方法用于释放许可。


#### 2\. 内部类 Sync 的实现


Semaphore 的核心实现依赖于其内部类 `Sync`，Sync 是 `AbstractQueuedSynchronizer` 的子类。根据是否是公平模式，有两种实现：`NonfairSync` 和 `FairSync`。


* **NonfairSync**：非公平模式。线程调用 `acquire` 时直接尝试获取许可，不保证顺序。
* **FairSync**：公平模式。线程获取许可遵循队列顺序。



```


|  | abstract static class Sync extends AbstractQueuedSynchronizer { |
| --- | --- |
|  | Sync(int permits) { |
|  | setState(permits);  // 设置初始许可数 |
|  | } |
|  |  |
|  | final int getPermits() { |
|  | return getState(); |
|  | } |
|  |  |
|  | final int nonfairTryAcquireShared(int acquires) { |
|  | for (;;) { |
|  | int available = getState(); |
|  | int remaining = available - acquires; |
|  | if (remaining < 0 || compareAndSetState(available, remaining)) |
|  | return remaining; |
|  | } |
|  | } |
|  |  |
|  | protected final boolean tryReleaseShared(int releases) { |
|  | for (;;) { |
|  | int current = getState(); |
|  | int next = current + releases; |
|  | if (compareAndSetState(current, next)) |
|  | return true; |
|  | } |
|  | } |
|  | } |


```

* **非公平模式（`nonfairTryAcquireShared`）**：直接尝试获取许可，如果成功则更新 `state`。
* **公平模式（`FairSync`）**：遵循 AQS 的队列顺序，确保 FIFO 访问。


### 三、Semaphore 的使用方法


Semaphore 提供了三种主要方法来操作许可：


* **acquire()**：获取一个许可，若没有许可则阻塞。
* **release()**：释放一个许可，唤醒等待的线程。
* **tryAcquire()**：尝试获取许可，但不阻塞。



```


|  | Semaphore semaphore = new Semaphore(3); // 初始许可数为 3 |
| --- | --- |
|  |  |
|  | // 获取许可，若无可用许可则阻塞 |
|  | semaphore.acquire(); |
|  |  |
|  | // 释放许可 |
|  | semaphore.release(); |
|  |  |
|  | // 尝试获取许可，若不可用立即返回 false，不会阻塞 |
|  | if (semaphore.tryAcquire()) { |
|  | // do something |
|  | } |


```

acquire() 源码



```


|  | public void acquire() throws InterruptedException { |
| --- | --- |
|  | sync.acquireSharedInterruptibly(1); |
|  | } |


```

`acquireSharedInterruptibly(1)` 会调用 `nonfairTryAcquireShared(1)` 或 `tryAcquireShared`（公平模式），尝试获取许可，若没有足够许可则阻塞等待。


release() 源码



```


|  | public void release() { |
| --- | --- |
|  | sync.releaseShared(1); |
|  | } |


```

`releaseShared(1)` 增加许可数并唤醒阻塞线程，使等待线程得以继续执行。


### 四、Semaphore 的应用场景


Semaphore 非常适合控制对有限资源的访问，典型的应用场景有：


#### 1\. 连接池限流


在数据库连接池中，可以使用 Semaphore 限制同时访问连接的线程数。例如，数据库连接数有限，通过 Semaphore 控制同时访问的线程数量，避免过度负载。



```


|  | public class DatabaseConnectionPool { |
| --- | --- |
|  | private static final int MAX_CONNECTIONS = 5; |
|  | private final Semaphore semaphore = new Semaphore(MAX_CONNECTIONS); |
|  |  |
|  | public Connection getConnection() throws InterruptedException { |
|  | semaphore.acquire(); |
|  | return acquireConnection(); |
|  | } |
|  |  |
|  | public void releaseConnection(Connection conn) { |
|  | releaseConnection(conn); |
|  | semaphore.release(); |
|  | } |
|  | } |


```

#### 2\. 控制线程并发数


在高并发任务中，Semaphore 可限制同时执行的线程数量，例如，控制下载任务的并发数，避免过多线程导致的系统负载过高。


#### 3\. 资源分配和限流


Semaphore 适合资源共享场景，如共享打印机、固定线程数的线程池等，确保资源不会被过度占用。


### 五、与其他 JUC 工具的对比


#### 1\. CountDownLatch


* **许可机制**：Semaphore 的许可数可以动态变化，而 CountDownLatch 的计数器只能递减。
* **适用场景**：Semaphore 控制并发资源访问数量，而 CountDownLatch 用于线程等待。


#### 2\. CyclicBarrier


* **作用**：Semaphore 适合控制资源访问并发数，而 CyclicBarrier 则用于线程间的同步点，使所有线程达到某个屏障时一起继续。
* **灵活性**：Semaphore 更灵活，可随时 `release()`，不必等待所有线程。


#### 3\. ReentrantLock


* **模式**：ReentrantLock 是排他锁，每次只允许一个线程访问。Semaphore 允许多个线程共享资源（多个许可）。
* **适用场景**：ReentrantLock 适合独占资源场景，Semaphore 适合资源池、限流等。


#### 4\. Phaser


* **复杂度**：Phaser 用于复杂的多阶段同步，可以动态增加/减少线程，而 Semaphore 主要用于固定数量资源控制。
* **适用性**：Phaser 适合分阶段任务同步，而 Semaphore 适合控制资源并发数。




---


庐山烟雨浙江潮，未至千般恨不消。


到得还来别无事，庐山烟雨浙江潮。


近半年不会再更新了，居家办公的日子要结束了。读书，世界就在眼前；不读书，眼前就是世界。与你共勉！


 本博客参考[豆荚加速器](https://yirou.org)。转载请注明出处！
