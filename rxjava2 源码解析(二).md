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
    这个调度器的源码注释是：任务在当前线程工作（不是线程池）但是不会立即执行，任务会被放入队列并在当前
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
发布线程在上一篇的文章已经提到，内部是一个worker，那么订阅线程也是么，很显然必须是的，接下来我们来看下源代码：
```java
// 还是从subscribeActul开始（原因见上一篇文章）
public void subscribeActual(Subscriber<? super T> s) {
    Worker worker = scheduler.createWorker();

    if (s instanceof ConditionalSubscriber) {
        source.subscribe(new ObserveOnConditionalSubscriber<T>(
                (ConditionalSubscriber<? super T>) s, worker, delayError, prefetch));
    } else {
        // 其内部封装了一个ObserveOnsubcriber，这是个对下流订阅者的封装，主要什么作用呢，为什么要这个呢，
        // 其实这个涉及订阅线程内部的机制，接着看源代码了解其内部机制
        source.subscribe(new ObserveOnSubscriber<T>(s, worker, delayError, prefetch));
    }
}
  // 基类
  abstract static class BaseObserveOnSubscriber<T>
  extends BasicIntQueueSubscription<T>
  implements FlowableSubscriber<T>, Runnable {
      private static final long serialVersionUID = -8241002408341274697L;

      final Worker worker;

      final boolean delayError;

      final int prefetch;

      final int limit;

      final AtomicLong requested;

      Subscription upstream;

      SimpleQueue<T> queue;

      volatile boolean cancelled;

      volatile boolean done;

      Throwable error;

      int sourceMode;

      long produced;

      boolean outputFused;

      BaseObserveOnSubscriber(
              Worker worker,
              boolean delayError,
              int prefetch) {
          this.worker = worker;
          this.delayError = delayError;
          this.prefetch = prefetch;
          this.requested = new AtomicLong();
          this.limit = prefetch - (prefetch >> 2);
      }

      @Override
      public final void onNext(T t) {
          if (done) {
              return;
          }
          if (sourceMode == ASYNC) {
              trySchedule();
              return;
          }
          /**
          * 当上游的装修者（上一篇提到的装修者模式）调用onNext方 * 法时，这时并没有类似的去调用下游的onNext方法，那这个
          * 时候其实就是订阅者线程模式的核心原理：采用queue队列
          * 进行数据的store，这里尝试将数据放进队列
          */
          if (!queue.offer(t)) {
              upstream.cancel();

              error = new MissingBackpressureException("Queue is full?!");
              done = true;
          }
          // 开启订阅者线程池模式的调度，具体实现在子类中实现
          trySchedule();
      }

      @Override
      public final void onError(Throwable t) {
          if (done) {
              RxJavaPlugins.onError(t);
              return;
          }
          error = t;
          done = true;
          trySchedule();
      }

      @Override
      public final void onComplete() {
          if (!done) {
              done = true;
              trySchedule();
          }
      }

      // 这里并没有向上传递request请求，而是把自己当做数据发射者进行request计数
      @Override
      public final void request(long n) {
          if (SubscriptionHelper.validate(n)) {
              BackpressureHelper.add(requested, n);
              // 开启调度
              trySchedule();
          }
      }

      @Override
      public final void cancel() {
          if (cancelled) {
              return;
          }

          cancelled = true;
          upstream.cancel();
          worker.dispose();

          if (getAndIncrement() == 0) {
              queue.clear();
          }
      }

      // 调度代码
      final void trySchedule() {
          // 上一篇文章讲过这个的用法
          if (getAndIncrement() != 0) {
              return;
          }
          // 启用一个work来进行任务的执行 this对象说明实现了runable接口
          worker.schedule(this);
      }

      // 调度实现的代码
      @Override
      public final void run() {
          if (outputFused) {
              runBackfused();
          } else if (sourceMode == SYNC) {
              runSync();
          } else {
              // 一般会调用runAsync方法
              runAsync();
          }
      }

      abstract void runBackfused();

      abstract void runSync();

      abstract void runAsync();

      final boolean checkTerminated(boolean d, boolean empty, Subscriber<?> a) {
          if (cancelled) {
              clear();
              return true;
          }
          if (d) {
              if (delayError) {
                  if (empty) {
                      cancelled = true;
                      Throwable e = error;
                      if (e != null) {
                          a.onError(e);
                      } else {
                          a.onComplete();
                      }
                      worker.dispose();
                      return true;
                  }
              } else {
                  Throwable e = error;
                  if (e != null) {
                      cancelled = true;
                      clear();
                      a.onError(e);
                      worker.dispose();
                      return true;
                  } else
                  if (empty) {
                      cancelled = true;
                      a.onComplete();
                      worker.dispose();
                      return true;
                  }
              }
          }

          return false;
      }

      @Override
      public final int requestFusion(int requestedMode) {
          if ((requestedMode & ASYNC) != 0) {
              outputFused = true;
              return ASYNC;
          }
          return NONE;
      }

      @Override
      public final void clear() {
          queue.clear();
      }

      @Override
      public final boolean isEmpty() {
          return queue.isEmpty();
      }
  }

  // 具体实现类
  static final class ObserveOnSubscriber<T> extends BaseObserveOnSubscriber<T>
  implements FlowableSubscriber<T> {

      private static final long serialVersionUID = -4547113800637756442L;

      final Subscriber<? super T> downstream;

      ObserveOnSubscriber(
              Subscriber<? super T> actual,
              Worker worker,
              boolean delayError,
              int prefetch) {
          super(worker, delayError, prefetch);
          this.downstream = actual;
      }

      //这是上游回调这个subscriber时调用的方法，详情见上一篇文章
      @Override
      public void onSubscribe(Subscription s) {
          if (SubscriptionHelper.validate(this.upstream, s)) {
              this.upstream = s;

              if (s instanceof QueueSubscription) {
                  @SuppressWarnings("unchecked")
                  QueueSubscription<T> f = (QueueSubscription<T>) s;

                  int m = f.requestFusion(ANY | BOUNDARY);

                  if (m == SYNC) {
                      sourceMode = SYNC;
                      queue = f;
                      done = true;

                      downstream.onSubscribe(this);
                      return;
                  } else
                  if (m == ASYNC) {
                      sourceMode = ASYNC;
                      queue = f;

                      downstream.onSubscribe(this);

                      s.request(prefetch);

                      return;
                  }
              }
              // 设置缓存队列
              // 这里涉及一个特别之处就是预获取（提前获取数据）
              queue = new SpscArrayQueue<T>(prefetch);
              // 触发下游subscriber 如果有request则会触发下游对上游数据的request
              downstream.onSubscribe(this);
              // 请求上游数据 上面的代码和这行代码就是起到承上启下的一个作用，也就是预获取，放在队列中
              s.request(prefetch);
          }
      }

      @Override
      void runSync() {
          int missed = 1;

          final Subscriber<? super T> a = downstream;
          final SimpleQueue<T> q = queue;

          long e = produced;

          for (;;) {

              long r = requested.get();

              while (e != r) {
                  T v;

                  try {
                      v = q.poll();
                  } catch (Throwable ex) {
                      Exceptions.throwIfFatal(ex);
                      cancelled = true;
                      upstream.cancel();
                      a.onError(ex);
                      worker.dispose();
                      return;
                  }

                  if (cancelled) {
                      return;
                  }
                  if (v == null) {
                      cancelled = true;
                      a.onComplete();
                      worker.dispose();
                      return;
                  }

                  a.onNext(v);

                  e++;
              }

              if (cancelled) {
                  return;
              }

              if (q.isEmpty()) {
                  cancelled = true;
                  a.onComplete();
                  worker.dispose();
                  return;
              }

              int w = get();
              if (missed == w) {
                  produced = e;
                  missed = addAndGet(-missed);
                  if (missed == 0) {
                      break;
                  }
              } else {
                  missed = w;
              }
          }
      }

      @Override
      void runAsync() {
          int missed = 1;

          final Subscriber<? super T> a = downstream;
          final SimpleQueue<T> q = queue;

          long e = produced;

          for (;;) {

              long r = requested.get();

              while (e != r) {
                  boolean d = done;
                  T v;

                  try {
                      // 获取数据
                      v = q.poll();
                  } catch (Throwable ex) {
                      Exceptions.throwIfFatal(ex);

                      cancelled = true;
                      upstream.cancel();
                      q.clear();

                      a.onError(ex);
                      worker.dispose();
                      return;
                  }

                  boolean empty = v == null;

                  if (checkTerminated(d, empty, a)) {
                      return;
                  }

                  if (empty) {
                      break;
                  }

                  a.onNext(v);

                  e++;
                  // limit = prefetch - (prefetch >> 2)
                  // prefetch  = BUFFER_SIZE（上一篇文章提到的默认128）
                  // 前面说过，订阅者把自己当成一个发射者，那数/据从哪里来呢，而且还要持续有数据，那么下面代码说明了数据来源，当数据达到limit，开始新的数据的prefetch，每次preftch的数量是limit
                  if (e == limit) {
                      if (r != Long.MAX_VALUE) {
                          r = requested.addAndGet(-e);
                      }
                      upstream.request(e);
                      e = 0L;
                  }
              }

              if (e == r && checkTerminated(done, q.isEmpty(), a)) {
                  return;
              }

              // 下面的代码机制在上一篇讲过主要涉及异步编程技巧
              int w = get();
              if (missed == w) {
                  produced = e;
                  missed = addAndGet(-missed);
                  if (missed == 0) {
                      break;
                  }
              } else {
                  missed = w;
              }
          }
      }

      @Override
      void runBackfused() {
          int missed = 1;

          for (;;) {

              if (cancelled) {
                  return;
              }

              boolean d = done;

              downstream.onNext(null);

              if (d) {
                  cancelled = true;
                  Throwable e = error;
                  if (e != null) {
                      downstream.onError(e);
                  } else {
                      downstream.onComplete();
                  }
                  worker.dispose();
                  return;
              }

              missed = addAndGet(-missed);
              if (missed == 0) {
                  break;
              }
          }
      }

      @Nullable
      @Override
      public T poll() throws Exception {
          T v = queue.poll();
          if (v != null && sourceMode != SYNC) {
              long p = produced + 1;
              if (p == limit) {
                  produced = 0;
                  upstream.request(p);
              } else {
                  produced = p;
              }
          }
          return v;
      }

  }
```
为何要将订阅者这样区别设置呢，其实原因很简单，**订阅者和发布者需要不同的线程机制异步地执行，比如订阅者需要computation的线程机制来进行大量的耗时数据计算，但又要保持一致的装修者模式，所以源码的做法是订阅者这边打破回调的调用流，采用数据队列进行两个线程池之间的数据传送**。
#### 本文总结
笔者喜欢总结，总结意味着我们反思和学习前面的知识点，应用点以及自身的不足。
  1. rxjava2线程调度的原理机制，不同场景下线程机制需要进行定制
  2. rxjava2生产和消费的异步原理和实现方式
