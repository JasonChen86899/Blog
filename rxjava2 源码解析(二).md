前一篇文章我们讲述到rxjava2 的内部设计模式与原理机制，包括观察者模式和装饰者模式，其本质上都是rxjava2的事件驱动，那么本篇文章将会讲到rxjava2的另外一个功能：异步功能

#### rxjava2 源码解析
依旧是阅读源码，还是从源码开始带着疑惑去读，前一篇文章我们讲到subcribeOn方法内部的实现涉及线程池：Scheduler.Worker w = scheduler.createWorker() 这边涉及两个重要组件：
1. scheduler调度器
2. 自定义线程池
##### scheduler调度器源码解析
```java
public final class Schedulers {
    @NonNull
    // 针对单一任务设置的单个定时线程池
    static final Scheduler SINGLE;

    @NonNull
    // 针对计算任务设置的定时线程池的资源池（数组）
    static final Scheduler COMPUTATION;

    @NonNull
    // 针对IO任务设置的单个可复用的定时线程池
    static final Scheduler IO;

    @NonNull
    // trampoline翻译是蹦床（佩服作者的脑洞）
    这个调度器的源码注释是：任务在当前线程工作但是不会立即执行，任务会被放入队列并在当前
    的任务完成之后执行。简单点说其实就是入队然后慢慢线性执行（这里巧妙的方法其实和前面我们所讲的回压实现机制基本是一致的，值得借鉴）
    static final Scheduler TRAMPOLINE;

    @NonNull
    // 单个的周期线程池和single基本一致唯一不同的是single对thread进行了一个简单的NonBlocking封装，这个封装从源码来看基本没有作用，只是一个marker interface标志接口
    static final Scheduler NEW_THREAD;
```
一共五种调度器，分别对应不同的场景，当然企业可以针对自身的场景设置自己的调度器
###### computation调度器源码分析
computation调度器针对大量计算场景，在后端并发场景会更多的用到，那么其是如何实现的呢？接下来带着疑惑进行源码分析
```java
  public final class ComputationScheduler extends Scheduler implements SchedulerMultiWorkerSupport {
    // 资源池
    final AtomicReference<FixedSchedulerPool> pool;

    // 这是computationScheduler类中实现的createWork()方法
    public Worker createWorker() {
      // 创建EventLoop工作者，入参是一个PoolWorker
        return new EventLoopWorker(pool.get().getEventLoop());
    }

  static final class FixedSchedulerPool implements SchedulerMultiWorkerSupport {
          final int cores;
          // 资源池工作者，每个工作者其实都是一个定时线程池
          final PoolWorker[] eventLoops;
          long n;
          // 对应前面的函数调用
          public PoolWorker getEventLoop() {
            int c = cores;
            if (c == 0) {
                return SHUTDOWN_WORKER;
            }
            // simple round robin, improvements to come
            // 这里其实就是从工作者数组中轮询选出一个工作者
            这里其实拥有提升和优化的空间，这里笔者可能会向开源社区提交一个pr
            以此进行比较好的调度器调度
            return eventLoops[(int)(n++ % c)];
          }
  // 此处是一个简单的封装        
  static final class PoolWorker extends NewThreadWorker {
          PoolWorker(ThreadFactory threadFactory) {
              super(threadFactory);
          }
      }

  public class NewThreadWorker extends Scheduler.Worker implements Disposable {
    private final ScheduledExecutorService executor;

    volatile boolean disposed;

    public NewThreadWorker(ThreadFactory threadFactory) {
        // 进行定时线程池的初始化
        executor = SchedulerPoolFactory.create(threadFactory);
    }

    public static ScheduledExecutorService create(ThreadFactory factory) {
      final ScheduledExecutorService exec =
      // 初始化一个定时线程池
      Executors.newScheduledThreadPool(1, factory);
      tryPutIntoPool(PURGE_ENABLED, exec);
      return exec;
    }
```
上述代码清晰的展示了computation调度器的实现细节，这里需要说明的是定时线程池的core设置为1，线程池的个数最多为cpu数量，这里涉及到ScheduledThreadPoolExecutor定时线程池的原理，简单的说起内部是一个可自动增长的数组（队列）类似于ArrayList，也就是说队列永远不会满，线程池中的线程数不会增加。  
接下来结合订阅线程和发布线程分析其之间如何进行沟通的本质
#### 本文总结
笔者喜欢总结，总结意味着我们反思和学习前面的知识点，应用点以及自身的不足。
* 设计模式：观察者模式和装修者模式
* 并发处理技巧：回压策略（其实本质是缓存）的实现原理以及细节点
