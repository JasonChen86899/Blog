ReactiveX 响应式编程库，这是一个程序库，通过使用**可观察的事件序列**来构成**异步**和**事件驱动**的程序。
其简化了异步多线程编程，在以前多线程编程的世界中，锁，可重入锁，同步队列器，信号量，并发同步器，同步计数器，
并行框架等都是具有一定的使用门槛，稍有不慎或者使用不成熟或对其源码理解不深入都会造成相应的程序错误和程序性能的低下。
#### 观察者模型
24种设计模式的一种，观察者Observer和主题Subject之间建立组合关系：Subject类实例中包含观察者Observer的引用，
增加引用的目的就是为了通知notify，重要点就是要在Subject的notify功能中调用Observer的接受处理函数receiveAndHandle
个人理解：观察者模型其实是一种异步回调通知，将数据的处理者先注册到数据的输入者那边，这样通过数据输入者执行某个函数
去调用数据处理者的某个处理方法。
#### RxJava2
Rx有很多语言的实现库，目前比较出名的就是RxJava2。这里主讲Rxjava2的部门源码解读，内部设计机制和内部执行的线程模型
#### 基本使用
使用rxjava2大致分为四个操作：
1. 建立数据发布者
2. 添加数据变换函数
3. 设置数据发布线程池机制，订阅线程池机制
4. 添加数据订阅者
```java
// 创建flowable
Flowable<Map<String, Map<String,Object>>> esFlowable = Flowable.create(new ElasticSearchAdapter(), BackpressureStrategy.BUFFER);
Disposable disposeable = esFlowable
    // map操作 1.采集、2.清洗
    .map(DataProcess::dataProcess)
    .subscribeOn(Schedulers.single())
    //计算任务调度器
    .observeOn(Schedulers.computation())
    // 订阅者 consumer 执行运算
    .subscribe(keyMaps -> new PredictEntranceForkJoin().predictLogic(keyMaps));
```

以上就是一个实际的例子，里面的ElasticSearchAdapter实际隐藏了一个用户自定义实现数据生产的subscribe接口：FlowableOnSubscribe<T> source
用户需要实现这个接口函数：void subscribe(@NonNull FlowableEmitter<T> emitter) throws Exception 这个接口主要用于内部回调，后面会有具体分析，
emitter 英文翻译发射器，很形象，数据就是由它产生的，也是业务系统需要对接的地方，一般业务代码实现这个接口类然后发射出需要处理的原始数据。
map函数作为数据变换处理的功能函数将原来的数据输入变换为另外的数据集合，然后设置发布的线程池机制subscribeOn(Schedulers.single())，订阅的线程池
机制observeOn(Schedulers.computation())，最后添加数据订阅函数，也就是业务系统需要实现另外一个地方，从而实现数据的自定义处理消费。
#### rxjava2支持的lambda语法
* 创建操作符：just fromArray empty error never fromIterable timer interval intervalRange range/rangeLong defer
* 变换操作符：map flatMap flatmapIterable concatMap switchmap cast scan buffer toList groupBy toMap
* 过滤操作符：filter take takeLast firstElement/lastElement first/last firstOrError/lastOrError elementAt/elementAtOrError ofType skip/skipLast
ignoreElements distinct/distinctUntilChanged timeout throttleFirst throttleLast/sample throttleWithTimeout/debounce
* 合并聚合操作符：startWith/startWithArray concat/concatArray merge/mergeArray concatDelayError/mergeDelayError zip combineLatest combineLatestDelayError
reduce count collect
* 条件操作符：all ambArray contains any isEmpty defaultIfEmpty switchIfEmpty sequenceEqual takeUntil takeWhile skipUntil skipWhile

有一篇博客详细介绍了rxjava的各种操作符，链接[https://maxwell-nc.github.io/android/rxjava2-1.html](https://maxwell-nc.github.io/android/rxjava2-1.html)
#### rxjava2 源码解析
阅读源码个人比较喜欢带着疑惑去看，这样与目标有方向。接下来的分析以Flowable为例，这里所有的例子都是按照Flowable为例，因为Flowable在实际
项目中比Observable可能用的多，因为实际场景中数据生产速度和数据消费速度都会有一定的不一致甚至数据生产速度远大于数据消费速度。
##### 数据发布和订阅
首先从数据订阅者开始，点进源码看进一步解析，里面有很多subscribe重载接口：
```java
  public final Disposable subscribe(Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING,
        Functions.EMPTY_ACTION, FlowableInternalHelper.RequestMax.INSTANCE);
  }
  public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
      Action onComplete, Consumer<? super Subscription> onSubscribe) {
    ObjectHelper.requireNonNull(onNext, "onNext is null");
    ObjectHelper.requireNonNull(onError, "onError is null");
    ObjectHelper.requireNonNull(onComplete, "onComplete is null");
    ObjectHelper.requireNonNull(onSubscribe, "onSubscribe is null");

    //组装成FlowableSubscriber
    LambdaSubscriber<T> ls = new LambdaSubscriber<T>(onNext, onError, onComplete, onSubscribe);
    //调用核心的订阅方法
    subscribe(ls);

    return ls;
  }
  public final void subscribe(FlowableSubscriber<? super T> s) {
      ObjectHelper.requireNonNull(s, "s is null");
      try {
          //注册一些钩子这里对此不进行讲解，主要不是核心方法
          Subscriber<? super T> z = RxJavaPlugins.onSubscribe(this, s);
          ObjectHelper.requireNonNull(z, "The RxJavaPlugins.onSubscribe hook returned a null FlowableSubscriber. Please check the handler provided to RxJavaPlugins.setOnFlowableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");
          //核心订阅方法，从名字也能读出是指订阅实际调用处
          //不同的数据产生类也就是实现Flowable抽象类的类
          //比如FlowableCreate，FlowSingle，FlowMap等等去实现自己的实际方法
          subscribeActual(z);
      } catch (NullPointerException e) { // NOPMD
          throw e;
      } catch (Throwable e) {
          Exceptions.throwIfFatal(e);
          // can't call onError because no way to know if a Subscription has been set or not
          // can't call onSubscribe because the call might have set a Subscription already
          RxJavaPlugins.onError(e);

          NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
          npe.initCause(e);
          throw npe;
      }
  }
  //选择FlowCreate的subscribeActual(Subscriber<? super T> t)方法进行剖析
  public void subscribeActual(Subscriber<? super T> t) {
      BaseEmitter<T> emitter;
      //根据不同的回压模式选择不一样的数据发射类
      //神奇的回压模式其实本质上就是一个个数据发射-消费模式
      switch (backpressure) {
      case MISSING: {
          emitter = new MissingEmitter<T>(t);
          break;
      }
      case ERROR: {
          emitter = new ErrorAsyncEmitter<T>(t);
          break;
      }
      case DROP: {
          emitter = new DropAsyncEmitter<T>(t);
          break;
      }
      case LATEST: {
          emitter = new LatestAsyncEmitter<T>(t);
          break;
      }
      default: {
          emitter = new BufferAsyncEmitter<T>(t, bufferSize());
          break;
      }
      }
      //回调注册的FlowableSubscriber的onSubscribe方法
      //这里非常重要，因为这里涉及了rxjava特有的 request请求再消费数据的模式
      //也就是说如果没有request数据，那么就不会调用数据发射（发布）者的onNext方法，
      //那么数据订阅者也就不会消费到数据
      t.onSubscribe(emitter);
      try {
          //回调注册的FlowableOnSubscribe<T> source的subscribe方法
          //这个source其实就是在创建Flow流时注册的数据产生类，进一步验证了上文中
          //提及的其需要实现FlowableOnSubscribe<T>接口
          source.subscribe(emitter);
      } catch (Throwable ex) {
          Exceptions.throwIfFatal(ex);
          emitter.onError(ex);
      }
  }
  //重点分析BufferAsyncEmitter这个类，看字面意思这是一个switch的默认选择类，
  //但其实它是回压策略为BUFFER时的数据发射类
  //首先这个类的构造函数具有两个参数，很明显这是 actul就是前面的t这个变量，也就是
  //注册的数据消费（订阅）者，capacityHint则是设置容量大小的，默认是128，如果需要扩大需要
  //自行设置环境变量 rx2.buffer-size
  BufferAsyncEmitter(Subscriber<? super T> actual, int capacityHint) {
      super(actual);
      this.queue = new SpscLinkedArrayQueue<T>(capacityHint);
      this.wip = new AtomicInteger();
  }

  public void onNext(T t) {
     if (done || isCancelled()) {
         return;
     }

     if (t == null) {
         onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
         return;
     }
     // queue 是存储元素的队列，也就是buffer的核心存储。
     // 当我们开始向下游发送数据的时候首先存入队列，然后下面的drain则是进行核心的
     queue.offer(t);
     drain();
  }
  //核心的类
  void drain() {
    //关键的地方 解决生产速率和消费速率不一致的关键地方，也是我们写并发程序值得借鉴的地方。
    //当数据的产生者（发布）频繁调用onNext方法时，这里产生并发调用关系，wip变量是atomic变量，
    //当第一次执行drain函数时，为0继续执行后面的流程，当快速的继续调用onNext方法时，wip不为0然后返回
    //那么后面的流程我们其实已经很大概率会猜测到应该是去取队列的数据然后做一些操作
   if (wip.getAndIncrement() != 0) {
       return;
   }

   int missed = 1;
   //这里的downstream其实就是注册的数据订阅者，它是基类BaseEmitter的变量，前面初始化时调用了基类的构造函数
   final Subscriber<? super T> a = downstream;
   final SpscLinkedArrayQueue<T> q = queue;

   for (;;) {
       long r = get();
       long e = 0L;

       while (e != r) {
           if (isCancelled()) {
               q.clear();
               return;
           }

           boolean d = done;
           //取队列中的数据
           T o = q.poll();

           boolean empty = o == null;

           if (d && empty) {
               Throwable ex = error;
               if (ex != null) {
                   error(ex);
               } else {
                   complete();
               }
               return;
           }

           if (empty) {
               break;
           }
           //此处回调订阅者的onNext方法去真正的执行数据实例程序
           //到此数据从产生到消费其生命周期已经走完
           a.onNext(o);

           e++;
       }

       if (e == r) {
           if (isCancelled()) {
               q.clear();
               return;
           }

           boolean d = done;

           boolean empty = q.isEmpty();

           if (d && empty) {
               Throwable ex = error;
               if (ex != null) {
                   error(ex);
               } else {
                   complete();
               }
               return;
           }
       }

       if (e != 0) {
          //标记已经消费的个数
           BackpressureHelper.produced(this, e);
       }
       //前面说过wip会原子性的增加，而且是每调用一次onNext增加一次
       //missed从其名解释是指错过的意思，个人理解是错过消费的数据个数，错过消费
       //的意思其实就是指没有进行a.onNext数据消费处理的数据
       missed = wip.addAndGet(-missed);
       if (missed == 0) {
          //如果没有错过的数据也就是全部都消费完那就跳出for循环
          //此处for循环方式和JUC源码中Doug Lea的做法都有类似之处
           break;
          }
      }
  }
```
##### 操作符与线程池机制原理剖析
首先在进行源码分析之前讲述一下一种模式：装饰者模式 24种模式中的一种，在java io源码包中广泛应用
简单的来说是与被装饰者具有相同接口父类同时又对被装饰者进行一层封装（持有被装饰者的引用），以此用来加上自身的特性。
回归主题，当我们使用操作符和线程池机制的时候做法都是在数据发布者后面进行相应的函数操作：
```java
Disposable disposeable = scheduleObservable
            .map(aLong -> dataAdapter.handlerDpti())
            .map(DataProcess::dataProcess)
            .subscribeOn(Schedulers.single())
```
那么为何这么做，接下来我们进行源码分析：
1. subscribeOn map 方法都在Flowable类中：
```java
public final <R> Flowable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new FlowableMap<T, R>(this, mapper));
    }
public final Flowable<T> subscribeOn(@NonNull Scheduler scheduler, boolean requestOn) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new FlowableSubscribeOn<T>(this, scheduler, requestOn));
}
```
这里是实例方法调用，传进了this对象这个很关键，这里其实就是我们前面提到的装修者模式，持有上游对象也就是数据源source的引用
以FlowableSubscribeOn为例进行分析，这个类经常会用到，因为其内部设置了线程池的机制所以在实际使用项目中会大量使用，那么是如何
做到线程池方式的呢？进一步利用源码进行分析：
2. 装饰者的内部代码分析
以subscribeOn 为例
```java
  //很明显 实现的抽象类其实是装修者抽象类
  public final class FlowableSubscribeOn<T> extends AbstractFlowableWithUpstream<T , T>

  // 这个在前面我们重点分析过这是实际订阅执行的类方法，其实也就是我们说的装饰方法，里面实现了每个类自己的特定“装修”方法
  @Override
  public void subscribeActual(final Subscriber<? super T> s) {
      // 获取订阅者，下一篇文章会重点讲述rxjava的线程池分配机制
      Scheduler.Worker w = scheduler.createWorker();
      final SubscribeOnSubscriber<T> sos = new SubscribeOnSubscriber<T>(s, w, source, nonScheduledRequests);
      // 跟前面一样调用数据订阅者的onSubscribe方法
      s.onSubscribe(sos);
      // 由分配的调度者进行订阅任务的执行
      w.schedule(sos);
  }

  // 开始分析SubscribeOnSubscriber这个静态内部类的内部代码
  // 实现了Runable用来异步执行
  static final class SubscribeOnSubscriber<T> extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable
    // 下游订阅引用
    final Subscriber<? super T> downstream;
    // 上游发射类引用
    final AtomicReference<Subscription> upstream;
    // 上游数据源引用 跟上游引用有区别，简单的说每个上游数据源引用有自己的上游发射类
    Publisher<T> source;
  // 这里是装饰的核心代码
  @Override
  public void run() {
      lazySet(Thread.currentThread());
      // source即为上游，表示其所装饰的源
      Publisher<T> src = source;
      source = null;
      // 调用上游的自身的subscribe方法，在上面一开始我们说这个方法内部会去调用自身实现的subscribeActual方法
      // 从而实现上游自己的特定方法，比如假设source是FlowCreate那么此处就会调用前面一开始我们所讲到的数据的发射
      src.subscribe(this);
  }

  // 既然已经保证了数据的发射那么数据的处理是不是也要处理
  // 很明显这是调用了下游订阅者的onNext方法
  @Override
  public void onNext(T t) {
      downstream.onNext(t);
  }
```
#### 本文总结
笔者喜欢总结，总结意味着我们反思和学习前面的知识点，应用点以及自身的不足。
* 设计模式：观察者模式和装修者模式
* 并发处理技巧：回压策略（其实本质是缓存）的实现原理以及细节点
