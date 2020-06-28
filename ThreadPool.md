

## 线程池

Java最初是没有提供线程池的。
而是现有线程，让我们假设一个场景，如果你需要执行一批任务，你需要创建很多线程来执行这一批任务。可是随着线程数的不断增加，我们发现对线程的管理越来越难，而且很多线程本身执行的时间还没有创建线程，创建线程时会产生系统开销，并且每个线程还会占用一定的内存等资源，更重要的是我们创建如此多的线程也会给稳定性带来危害，因为每个系统中，可创建线程的数量是有一个上限的，不可能无限的创建。线程执行完需要被回收，大量的线程又会给垃圾回收带来压力。但我们的任务确实非常多，如果都在主线程串行执行，那效率也太低了，那应该怎么办呢？于是便诞生了线程池来平衡线程与系统资源之间的关系。

我们来总结下如果每个任务都创建一个线程会带来哪些问题：

第一点，反复创建线程系统开销比较大，每个线程创建和销毁都需要时间，如果任务比较简单，那么就有可能导致创建和销毁线程消耗的资源比线程执行任务本身消耗的资源还要大。
第二点，过多的线程会占用过多的内存等资源，还会带来过多的上下文切换，同时还会导致系统不稳定

针对上面的两点问题，线程池有两个解决思路。

首先，针对反复创建线程开销大的问题，线程池用一些固定的线程一直保持工作状态并反复执行任务。

其次，针对过多线程占用太多内存资源的问题，解决思路更直接，线程池会根据需要创建线程，控制线程的总数量，避免占用过多内存资源。

线程池就好像是一个工厂，工厂里的工人数量是有限可控的，但是工厂的订单时不可控的。加入工厂车间内有5个工人，这个时候有10个订单，那这5个工人每人分配一个订单，做完了在做一个订单就好了。如果此时来了大量的订单，工厂会将这些订单放到一个待生产任务队列中。工人从这些任务队列中取订单任务执行就好了。如果继续来了任务，工厂会临时招聘一些员工来执行这些任务，待这些任务执行完毕后，经过一段时间的观察后，工厂根据自己的订单情况来决定是否要解聘这些临时招聘的工人，如果招聘了工人后还不能满足产能，那工厂就会拒绝接受订单。

## 线程池的工作原理

### 参数

## ![屏幕快照 2020-06-28 上午10.24.06](/Users/meiliwu/Desktop/屏幕快照 2020-06-28 上午10.24.06.png)

### 创建流程

![屏幕快照 2020-06-28 上午10.27.27](/Users/meiliwu/Desktop/屏幕快照 2020-06-28 上午10.27.27.png)

如上图所示，当向线程池体检任务时，线程池首先会检测核心线程池是否已满，如果核心线程池未满，则创建核心线程执行该任务，如果核心线程池已满，则检测任务队列是否已满，如果没满，则将任务加入到任务队列中，如果任务队列已满，则判断线程池是否已满，如果未满，则创建新的非核心线程执行任务，如果线程池已满。则按照拒绝策略来拒绝新进来的任务。

通过对流程图的创建流程分析，我们总结出线程池的几个特点。

线程池希望保持较少的线程数，并且只有在负载变得很大时才增加线程。

线程池只有在任务队列填满时才创建多于 corePoolSize 的线程，如果使用的是无界队列（例如 LinkedBlockingQueue），那么由于队列不会满，所以线程数不会超过 corePoolSize。

通过设置 corePoolSize 和 maxPoolSize 为相同的值，就可以创建固定大小的线程池。

通过设置 maxPoolSize 为很高的值，例如 Integer.MAX_VALUE，就可以允许线程池创建任意多的线程。

### corePoolSize

线程中核心线程池的数量，当没有任务执行时，这类线程也不会被移除

### maxPoolSize

线程池中线程存在的最大数量==核心线程数量+非核心线程数量

### keepAliveTime + TimeUnit

线程池中没有任务执行时，非核心线程的存活时间。当超过设定的时间 后，线程池会将非核心线程回收以回收系统资源。

### ThreadFactory

线程工厂，线程池用来创建新线程的类，我们可以自定义以给新创建的线程定义我们需要的名称

### WorkQueue

任务队列，

### Handler

线程池的拒绝策略。当传入超过线程池的工作能力是会调用



## 线程池的4中自带的拒绝策略

​		

```
public interface RejectedExecutionHandler {

    /**
     * Method that may be invoked by a {@link ThreadPoolExecutor} when
     * {@link ThreadPoolExecutor#execute execute} cannot accept a
     * task.  This may occur when no more threads or queue slots are
     * available because their bounds would be exceeded, or upon
     * shutdown of the Executor.
     *
     * <p>In the absence of other alternatives, the method may throw
     * an unchecked {@link RejectedExecutionException}, which will be
     * propagated to the caller of {@code execute}.
     *
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```



改接口有4个实现类

![image-20200628104335977](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200628104335977.png)

其中第一个AbortPolicy 是线程池默认的拒绝策略。

### AbortPolicy

改策略会在线程池工作饱和后直接丢掉新添加的工作任务，直接抛出一个类型为 RejectedExecutionException 的 RuntimeException，让你感知到任务被拒绝了，于是你便可以根据业务逻辑选择重试或者放弃提交等策略。

### DiscardPolicy

正如名字一样，这种策略会直接丢弃新添加的任务。不会改调用者任何提示。会造成数据丢失的风险

### DiscardOldestPolicy

和DiscardPolicy 一样，区别是DiscardPolicy 会丢弃新添加的工作任务。而DiscardOldestPolicy会丢弃工作队列对头的任务。同理也存在数据丢失的风险

### CallerRunsPolicy

当有新任务提交后，如果线程池没被关闭且没有能力执行，则把这个任务交于提交任务的线程执行，也就是谁提交任务，谁就负责执行任务。这样做主要有两点好处。

- 第一点新提交的任务不会被丢弃，这样也就不会造成业务损失。
- 第二点好处是，由于谁提交任务谁就要负责执行任务，这样提交任务的线程就得负责执行任务，而执行任务又是比较耗时的，在这段期间，提交任务的线程被占用，也就不会再提交新的任务，减缓了任务提交的速度，相当于是一个负反馈。在此期间，线程池中的线程也可以充分利用这段时间来执行掉一部分任务，腾出一定的空间，相当于是给了线程池一定的缓冲期。



当然我们也可以实现RejectedExecutionHandler 接口实现我们自定义的拒绝策略。



## 几种常见的线程池

1. FixedThreadPool
2. CachedThreadPool
3. ScheduledThreadPool
4. SingleThreadExecutor
5. SingleThreadScheduledExecutor
6. ForkJoinPool（Java8 添加）

### FixedThreadPool

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

从名字可以看出这类线程池的线程数量是固定的。这种线程池中核心线程数量==最大线程数量

它的特点是线程池中的线程数除了初始阶段需要从 0 开始增加外，之后的线程数量就是固定的，就算任务数超过线程数，线程池也不会再创建更多的线程来处理任务，而是会把超出线程处理能力的任务放到任务队列中进行等待。而且就算任务队列满了，到了本该继续增加线程数的时候，由于它的最大线程数和核心线程数是一样的，所以也无法再增加新的线程了。

### CachedThreadPool

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

这种线程池的线程最大数量是Integer.MAX_VALUE 也就是2^31-1，(这个数非常大，所以基本不可能达到），而当线程闲置时还可以对线程进行回收。也就是说该线程池的线程数量不是固定不变的，当然它也有一个用于存储提交任务的队列，但这个队列是 SynchronousQueue，队列的容量为0，实际不存储任何任务，它只负责对任务进行中转和传递，所以效率比较高。

### ScheduledThreadPool

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

第三个线程池是 ScheduledThreadPool，它支持定时或周期性执行任务。比如每隔 10 秒钟执行一次任务，而实现这种功能的方法主要有 3 种，如代码所示

```
ScheduledExecutorService service = Executors.newScheduledThreadPool(10);

service.schedule(new Task(), 10, TimeUnit.SECONDS);

service.scheduleAtFixedRate(new Task(), 10, 10, TimeUnit.SECONDS);

service.scheduleWithFixedDelay(new Task(), 10, 10, TimeUnit.SECONDS);
```

那么这 3 种方法有什么区别呢？

- 第一种方法 schedule 比较简单，表示延迟指定时间后执行一次任务，如果代码中设置参数为 10 秒，也就是 10 秒后执行一次任务后就结束。
- 第二种方法 scheduleAtFixedRate 表示以固定的频率执行任务，它的第二个参数 initialDelay 表示第一次延时时间，第三个参数 period 表示周期，也就是第一次延时后每次延时多长时间执行一次任务。
- 第三种方法 scheduleWithFixedDelay 与第二种方法类似，也是周期执行任务，区别在于对周期的定义，之前的 scheduleAtFixedRate 是以任务开始的时间为时间起点开始计时，时间到就开始执行第二次任务，而不管任务需要花多久执行；而 scheduleWithFixedDelay 方法以任务结束的时间为下一次循环的时间起点开始计时。



举个例子，假设某个同学正在熬夜写代码，需要喝咖啡来提神，假设每次喝咖啡都需要花10分钟的时间，如果此时采用第2种方法 scheduleAtFixedRate，时间间隔设置为 1 小时，那么他将会在每个整点喝一杯咖啡，以下是时间表：

00:00: 开始喝咖啡

00:10: 喝完了

01:00: 开始喝咖啡

01:10: 喝完了

02:00: 开始喝咖啡

02:10: 喝完了

但是假设他采用第3种方法 scheduleWithFixedDelay，时间间隔同样设置为 1 小时，那么由于每次喝咖啡需要10分钟，而 scheduleWithFixedDelay 是以任务完成的时间为时间起点开始计时的，所以第2次喝咖啡的时间将会在1:10，而不是1:00整，以下是时间表：

00:00: 开始喝咖啡

00:10: 喝完了

01:10: 开始喝咖啡

01:20: 喝完了

02:20: 开始喝咖啡

02:30: 喝完了

可见下次任务开始的时间是以第一次任务结束的时间+延迟时间。

注意：ScheduledThreadPool执行周期任务时，如果某次任务执行失败了，后续的任务就不会执行了。所以 ScheduledThreadPoolExecutor的最优实践：将所有执行代码用try-catch包裹。在catch块里再次提交任务。

### SingleThreadExecutor

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

第四种线程池是 SingleThreadExecutor，它会使用唯一的线程去执行任务，原理和 FixedThreadPool 是一样的，只不过这里线程只有一个，如果线程在执行任务的过程中发生异常，线程池也会重新创建一个线程来执行后续的任务。这种线程池由于只有一个线程，所以非常适合用于所有任务都需要按被提交的顺序依次执行的场景，而前几种线程池不一定能够保障任务的执行顺序等于被提交的顺序，因为它们是多线程并行执行的。

### SingleThreadScheduledExecutor

第五个线程池是 SingleThreadScheduledExecutor，它实际和第三种 ScheduledThreadPool 线程池非常相似，它只是 ScheduledThreadPool 的一个特例，内部只有一个线程，如源码所示：

```
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
    return new DelegatedScheduledExecutorService
        (new ScheduledThreadPoolExecutor(1));
}
```

### ForkJoinPool

没怎么用过呢。

### 为什么不应该自动创建线程池

自动创建线程池就是直接调用 Executors 的各种方法来生成前面学过的常见的线程池，例如 Executors.newCachedThreadPool()。但这样做是有一定风险的，接下来我们就来逐一分析自动创建线程池可能带来哪些问题。

### FixedThreadPool	

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

可以看出创建FixedThreadPool，系统使用了一个LinkedBlockingQueue队列，这种队列是没有长度现在，这就会导致一个问题，我们可以无限制的向线程池提交任务。如果我们对任务的处理速度比较慢，那么随着请求的增多，队列中堆积的任务也会越来越多，最终大量堆积的任务会占用大量内存，并发生 OOM ，也就是OutOfMemoryError，这几乎会影响到整个程序，会造成很严重的后果。

### SingleThreadExecutor

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

newSingleThreadExecutor 和 newFixedThreadPool 的原理是一样的，只不过把核心线程数和最大线程数都直接设置成了 1，但是任务队列仍是无界的 LinkedBlockingQueue，所以也会导致同样的问题，也就是当任务堆积时，可能会占用大量的内存并导致 OOM。

### CachedThreadPool

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

 CachedThreadPool 和前两种线程池不一样的地方在于任务队列使用的是 SynchronousQueue，SynchronousQueue 本身并不存储任务，而是对任务直接进行转发，这本身是没有问题的，但你会发现构造函数的第二个参数被设置成了 Integer.MAX_VALUE，这个参数的含义是最大线程数，所以由于 CachedThreadPool 并不限制线程的数量，当任务数量特别多的时候，就可能会导致创建非常多的线程，最终超过了操作系统的上限而无法创建新线程，或者导致内存不足

### ScheduledThreadPool 和 SingleThreadScheduledExecutor

```
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```

通过源码可以看出，它采用的任务队列是 DelayedWorkQueue，这是一个延迟队列，同时也是一个无界队列，所以和 LinkedBlockingQueue 一样，如果队列中存放过的任务，就可能导致 OOM。

可以看到，这几种自动创建的线程池都存在风险，相比较而言，我们自己手动创建会更好，因为我们可以更加明确线程池的运行规则，不仅可以选择适合自己的线程数量，更可以在必要的时候拒绝新任务的提交，避免资源耗尽的风险。



## 合适的线程数量是多少？CPU 核心数和线程数的关系

从上面的分析可知，我们不应该自动创建线程池，即不应该调用Executors的自带方法创建常用的线程池，而应该手动的创建线程池。即调用ThreadPoolExecutor的构造方法手动创建线程池，这就需要我们手动指定各个参数。

那么我们到底需要创建多少个核心线程，线程池的最大线程数量又该如何设定呢。

我们需要根据我们的任务性质设置不同的线程数量

### CPU 密集型任务

首先，我们来看 CPU 密集型任务，比如加密、解密、压缩、计算等一系列需要大量耗费 CPU 资源的任务。对于这样的任务最佳的线程数为 CPU 核心数的 1~2 倍，如果设置过多的线程数，实际上并不会起到很好的效果。此时假设我们设置的线程数量是 CPU 核心数的 2 倍以上，因为计算任务非常重，会占用大量的 CPU 资源，所以这时 CPU 的每个核心工作基本都是满负荷的，而我们又设置了过多的线程，每个线程都想去利用 CPU 资源来执行自己的任务，这就会造成不必要的上下文切换，此时线程数的增多并没有让性能提升，反而由于线程数量过多会导致性能下降。

### 耗时 IO 型任务  

第二种任务是耗时 IO 型，比如数据库、文件的读写，网络通信等任务，这种任务的特点是并不会特别消耗 CPU 资源，但是 IO 操作很耗时，总体会占用比较多的时间。对于这种任务最大线程数一般会大于 CPU 核心数很多倍，因为 IO 读写速度相比于 CPU 的速度而言是比较慢的，如果我们设置过少的线程数，就可能导致 CPU 资源的浪费。而如果我们设置更多的线程数，那么当一部分线程正在等待 IO 的时候，它们此时并不需要 CPU 来计算，那么另外的线程便可以利用 CPU 去执行其他的任务，互不影响，这样的话在任务队列中等待的任务就会减少，可以更好地利用资源

《Java并发编程实战》的作者 Brain Goetz 推荐的计算方法：

```
线程数 = CPU 核心数 *（1+平均等待时间/平均工作时间）
```

通过这个公式，我们可以计算出一个合理的线程数量，如果任务的平均等待时间长，线程数就随之增加，而如果平均工作时间长，也就是对于我们上面的 CPU 密集型任务，线程数就随之减少。

 太少的线程数会使得程序整体性能降低，而过多的线程也会消耗内存等其他资源，所以如果想要更准确的话，可以进行压测，监控 JVM 的线程情况以及 CPU 的负载情况，根据实际情况衡量应该创建的线程数，合理并充分利用资源。

结论     
综上所述我们就可以得出一个结论：

- 线程的平均工作时间所占比例越高，就需要越少的线程；

- 线程的平均等待时间所占比例越高，就需要越多的线程；


针对不同的程序，进行对应的实际测试就可以得到最合适的选择。

## 如何正确的关闭线程池

ThreadPoolExecutor提供了5个有关关闭线程池的方法

- void shutdown();
- boolean isShutdown();
- boolean isTerminated();
- boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
- List<Runnable> shutdownNow();

### shutdown

该方法用于安全地关闭一个线程池，调用该方法后线程池并不会立即关闭。因为这个时候线程池可能还有任务在执行。或者任务队列中有任务待执行，调用该方法后线程池会将所有的任务执行完毕后才关闭，但是调用改方法后线程池将不再接受新的工作任务，线程池则会根据拒绝策略直接拒绝后续新提交的任务

### isShutdown

它可以返回 true 或者 false 来判断线程池是否已经开始了关闭工作，也就是是否执行了 shutdown 或者 shutdownNow 方法。这里需要注意，如果调用 isShutdown() 方法的返回的结果为 true 并不代表线程池此时已经彻底关闭了，这仅仅代表线程池开始了关闭的流程，也就是说，此时可能线程池中依然有线程在执行任务，队列里也可能有等待被执行的任务。

也就是说该方法返回true，只是表示该线程池开启了关闭操作，但是可能还没有彻底关闭，只是一个标志位

### isTerminated

不同于isShutDown方法，isShutdown方法返回true，线程池也可能没有真正关闭，线程池会将任务队列中的任务执行完毕后才真正关闭。这个时候isTerminated回放回false，但是当线程池真正关闭后，isTerminated 返回true。

### awaitTermination

该方法不是用来关闭线程池的，而是主要用来判断线程池状态的。比如我们给 awaitTermination 方法传入的参数是 10 秒，那么它就会陷入 10 秒钟的等待，直到发生以下三种情况之一：

- 等待期间（包括进入等待状态之前）线程池已关闭并且所有已提交的任务（包括正在执行的和队列中等待的）都执行完毕，相当于线程池已经“终结”了，方法便会返回 true；
- 等待超时时间到后，第一种线程池“终结”的情况始终未发生，方法返回 false；
- 等待期间线程被中断，方法会抛出 InterruptedException 异常。
  也就是说，调用 awaitTermination 方法后当前线程会尝试等待一段指定的时间，如果在等待时间内，线程池已关闭并且内部的任务都执行完毕了，也就是说线程池真正“终结”了，那么方法就返回 true，否则超时返回 fasle。

我们则可以根据 awaitTermination() 返回的布尔值来判断下一步应该执行的操作。

### shutdownNow

它与第一种 shutdown 方法不同之处在于名字中多了一个单词 Now，也就是表示立刻关闭的意思。在执行 shutdownNow 方法之后，首先会给所有线程池中的线程发送 interrupt 中断信号，尝试中断这些任务的执行，然后会将任务队列中正在等待的所有任务转移到一个 List 中并返回，我们可以根据返回的任务 List 来进行一些补救的操作，例如记录在案并在后期重试。shutdownNow() 的源码如下所示。

    public List<Runnable> shutdownNow() { 
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try { 
              checkShutdownAccess();
              advanceRunState(STOP);
              interruptWorkers();
              tasks = drainQueue();
          } finally { 
              mainLock.unlock();
          } 
    
          tryTerminate();
          return tasks;
    }


你可以看到源码中有一行 interruptWorkers() 代码，这行代码会让每一个已经启动的线程都中断，这样线程就可以在执行任务期间检测到中断信号并进行相应的处理，提前结束任务。这里需要注意的是，由于 Java 中不推荐强行停止线程的机制的限制，即便我们调用了 shutdownNow 方法，如果被中断的线程对于中断信号不理不睬，那么依然有可能导致任务不会停止。可见我们在开发中落地最佳实践是很重要的，我们自己编写的线程应当具有响应中断信号的能力，应当利用中断信号来协同工作。

## 线程复用原理

我们知道线程池会使用固定数量或可变数量的线程来执行任务，但无论是固定数量或可变数量的线程，其线程数量都远远小于任务数量，面对这种情况线程池可以通过线程复用让同一个线程去执行不同的任务，那么线程复用背后的原理是什么呢？

线程池可以把线程和任务进行解耦，线程归线程，任务归任务，摆脱了之前通过 Thread 创建线程时的一个线程必须对应一个任务的限制。在线程池中，同一个线程可以从 BlockingQueue 中不断提取新任务来执行，其核心原理在于线程池对 Thread 进行了封装，并不是每次执行任务都会调用 Thread.start() 来创建新线程，而是让每个线程去执行一个“循环任务”，在这个“循环任务”中，不停地检查是否还有任务等待被执行，如果有则直接去执行这个任务，也就是调用任务的 run 方法，把 run 方法当作和普通方法一样的地位去调用，相当于把每个任务的 run() 方法串联了起来，所以线程数量并不增加。

再看一下线程池的工作原理：

![屏幕快照 2020-06-28 上午10.27.27](/Users/meiliwu/Desktop/屏幕快照 2020-06-28 上午10.27.27.png)

我们看一下execute方法

```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 step
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

```
 if (command == null)
        throw new NullPointerException();
```

首先进行入参判断，如果提交的任务为null，则抛出一个NPE。

接下来判断线程数量是否小于核心线程数量。如果线程池中线程数量小于核心线程数，则调用addWorker()方法新增一个worker，worker可以理解为一个线程。

```
int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
```



addWorker 方法的主要作用是在线程池中创建一个线程并执行第一个参数传入的任务，它的第二个参数是个布尔值，如果布尔值传入 true 代表增加线程时判断当前线程是否少于 corePoolSize，小于则增加新线程，大于等于则不增加；同理，如果传入 false 代表增加线程时判断当前线程是否少于 maxPoolSize，小于则增加新线程，大于等于则不增加，所以这里的布尔值的含义是以核心线程数为界限还是以最大线程数为界限进行是否新增线程的判断。addWorker() 方法如果返回 true 代表添加成功，如果返回 false 代表添加失败。

接下来

```
if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
```

如果代码执行到这里说明当前线程池中的线程数量大于或者等于核心线程数量，或者addWorker失败了。

此时就需要使用if (isRunning(c) && workQueue.offer(command)) 来检查线程池是否在运行中，如果线程池状态是 Running 就把任务放入任务队列中，也就是 workQueue.offer(command)。

如果线程池已经不再运行状态了。需要将刚添加到任务队列的任务移出队列并执行拒绝策略。

代码如下

```
if (! isRunning(recheck) && remove(command))
            reject(command);
```

接下来

```
else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
```

如果执行到这里说明线程池依旧在运行中。那么当任务被添加进来之后就需要防止没有可执行线程的情况发生（比如之前的线程被回收了或意外终止了），所以此时如果检查当前线程数为 0，也就是 workerCountOf**(recheck) == 0，那就执行 addWorker() 方法新建线程

再看一下最后一个else

```
 else if (!addWorker(command, false))
        reject(command);
```

执行到这里说明线程池已经不是running状态了或者任务队列已满。根据规则，此时线程池会创建新的非核心线程来执行任务。直到达到最大线程数。所以此时调用addWorker(command，false)来创建一个非核心线程。如果没有创建成功，说明线程数已满。此时执行拒绝策略 reject(command);

```
 final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
```



在 execute 方法中，多次调用 addWorker 方法把任务传入，addWorker 方法会添加并启动一个 Worker，这里的 Worker 可以理解为是对 Thread 的包装，Worker 内部有一个 Thread 对象，它正是最终真正执行任务的线程，所以一个 Worker 就对应线程池中的一个线程，addWorker 就代表增加线程。线程复用的逻辑实现主要在 Worker 类中的 run 方法里执行的 runWorker 方法中，简化后的 runWorker 方法代码如下所示。

addWorker方法的源码

```
 private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

```
 w = new Worker(firstTask);
            final Thread t = w.thread;
```

使用传递进来的runnable构建一个Worker对象，然后取出该对象的Thread对象。

```
 private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
      /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        
         Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

			 /** Delegates main run loop to outer runWorker. */
        public void run() {
            runWorker(this);
        }
    }
```

从源码可以看出Worker就是一个线程实现了runnable接口，内不持有一个Thread对象。

在Worker的run方法中我们可以看出调用了runWorker方法。

```
runWorker(Worker w) {
    Runnable task = w.firstTask;
    while (task != null || (task = getTask()) != null) {
        try {
            task.run();
        } finally {
            task = null;
        }
    }
}
```

可以看出，实现线程复用的逻辑主要在一个不停循环的 while 循环体中。

1. 通过取 Worker 的 firstTask 或者通过 getTask 方法从 workQueue 中获取待执行的任务。
2. 直接调用 task 的 run 方法来执行具体的任务（而不是新建线程）。

在这里，我们找到了最终的实现，通过取 Worker 的 firstTask 或者 getTask方法从 workQueue 中取出了新任务，并直接调用 Runnable 的 run 方法来执行任务，也就是如之前所说的，每个线程都始终在一个大循环中，反复获取任务，然后执行任务，从而实现了线程的复用。