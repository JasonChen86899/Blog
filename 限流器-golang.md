### 限流器  
顾名思义用来对高并发的请求进行流量限制的组件。  
为啥需要进行流量限制。首先后端服务由于各个业务的不同和复杂性，各自在容器部署的时候都可能会有单台的瓶颈，超过瓶颈会导致内存或者cpu的瓶颈，进而导致发生服务不可用或者单台容器直接挂掉或重启。
流量的限制在众多微服务和service mesh系统中多有应用。笔者在本文的程序示例均以golang作为示例。
#### 信号量限流
信号量 在众多开发语言中都会有相关信号量的设计。笔者在阅读一些语言开源实现后，总结出信号量的主要有非阻塞和阻塞两种。
信号量两个重要方法 Acquire() 和 Release()  
1. 非阻塞方式  
以并发安全的计数方式比如采用原子atomic加减进行
2. 阻塞方式  
采用锁或者阻塞队列方式，比如java JUC包中的信号量。以golang为示例如下
```go
// 采用channel作为底层数据结构，从而达到阻塞的获取和使用信号量
type Semaphore struct {
	innerChan chan struct{}
}
// 初始化信号量，本质初始化一个channel，channel的初始化大小为 信号量数值
func NewSemaphore(num uint64) *Semaphore {
	return &Semaphore{
		innerChan: make(chan struct{}, num),
	}
}
// 获取信号量，本质是 向channel放入元素，如果同时有很多协程并发获取信号量，则channel则会full阻塞，从而达到控制并发协程数的目的，也即是信号量的控制
func (s *Semaphore) Acquire() {
	for {
		select {
		case s.innerChan <- struct{}{}:
			return
		default:
			log.Error("semaphore acquire is blocking")
			time.Sleep(100 * time.Millisecond)
		}
	}
}
// 释放信号量 本质是 从channel中获取元素，由于有acquire的放入元素，所以此处一定能回去到元素 也就能释放成功，default只要是出于安全编程的目的
func (s *Semaphore) Release() {
	select {
	case <-s.innerChan:
		return
	default:
		return
	}
}
```

### 限流算法
主流的限流算法分为两种 **漏桶算法**和**令牌桶算法**，关于这两个算法有很多文章和论文都给出了详细的讲解。从原理上看，令牌桶算法和漏桶算法是相反的，一个“进水”，一个是“漏水”。值得一提的是Google Guava开源和Uber开源限流组件均采用漏桶算法。

所以此处笔者开门见山，直接展示此算法的golang的实现，代码如下
```go
// 此处截取自研的熔断器代码中的限流实现，这是非阻塞的实现
func (sp *servicePanel) incLimit() error {
	// 如果大于限制的条件则返回错误
	if sp.currentLimitCount.Load() > sp.currLimitFunc(nil) {
		return ErrCurrentLimit
	}
	sp.currentLimitCount.Inc()
	return nil
}

func (sp *servicePanel) clearLimit() {
	// 定期每秒重置计数器，从而达到每秒限制的并发数
	// 比如限制1000req/s，在这里指每秒清理1000的计数值
// 令牌桶是定期放，这里是逆思维，每秒清空，实现不仅占用内存低而且效率高
	t := time.NewTicker(time.Second)
	for {
		select {
		case <-t.C:
			sp.currentLimitCount.Store(0)
		}
	}
}
```
分析
上述的实现实际是比较粗糙的实现，没有严格按照每个请求方按照某个固定速率进行，而是以秒为单位，粗粒度的进行计数清零，这其实会造成某个瞬间双倍的每秒限流个数，虽然看上去不满足要求，但是在这个瞬间其实是只是一个双倍值，正常系统都应该会应付一瞬间双倍限流个数的请求量。
改进
如果要严格的按照每个请求按照某个固定数值进行，那么可以改进时间的粗力度，具体做法如下
```go
func (sp *servicePanel) incLimit() error {
	// 如果大于1则返回错误
	if sp.currentLimitCount.Load() > 1 {
		return ErrCurrentLimit
	}
	sp.currentLimitCount.Inc()
	return nil
}

func (sp *servicePanel) clearLimit() {
	// 1s除以每秒限流个数
	t := time.NewTicker(time.Second/time.Duration(sp.currLimitFunc(nil)))
	for {
		select {
		case <-t.C:
			sp.currentLimitCount.Store(0)
		}
	}
}
```

#### uber 开源实现RateLimit 源码解析
##### 引入方式
第一版本
go get github.com/uber-go/ratelimit@v0.1.0

第二版本
go get github.com/uber-go/ratelimit@master

首先强调一点，跟笔者自研的限流器最大的不同的是，这是一个阻塞调用者的限流组件。先不多说，开始讲解源码
Test 示例
```go
func ExampleRatelimit() {
	rl := ratelimit.New(100) // per second

	prev := time.Now()
	for i := 0; i < 10; i++ {
		now := rl.Take()
		if i > 0 {
			fmt.Println(i, now.Sub(prev))
		}
		prev = now
	}

	// Output:
	// 1 10ms
	// 2 10ms
	// 3 10ms
	// 4 10ms
	// 5 10ms
	// 6 10ms
	// 7 10ms
	// 8 10ms
	// 9 10ms
}
```
概念介绍
限流速率一般表示为 rate/s 即一秒内rate个请求

源码讲解
构造限流器
首先是构造一个Limiter 里面有一个perRequest 这是关键的一个变量，表示每个请求之间相差的间隔时间，这是此组件的算法核心思想，也就是说将请求排队，一秒之内有rate个请求，将这些请求排队，挨个来，每个请求的间隔就是1s/rate 从来达到 1s内rate个请求的概念，从而达到限流的目的。
```go
// New returns a Limiter that will limit to the given RPS.
func New(rate int, opts ...Option) Limiter {
	l := &limiter{
		perRequest: time.Second / time.Duration(rate),
		maxSlack:   -10 * time.Second / time.Duration(rate),
	}
	for _, opt := range opts {
		opt(l)
	}
	if l.clock == nil {
		l.clock = clock.New()
	}
	return l
}
```

限流器Take() 阻塞方法
Take() 方法 每次请求前使用，用来获取批准 返回批准时刻的时间

第一版本
```go
// Take blocks to ensure that the time spent between multiple
// Take calls is on average time.Second/rate.
func (t *limiter) Take() time.Time {
	t.Lock()
	defer t.Unlock()

	now := t.clock.Now()

	// If this is our first request, then we allow it.
	if t.last.IsZero() {
		t.last = now
		return t.last
	}

	// sleepFor calculates how much time we should sleep based on
	// the perRequest budget and how long the last request took.
	// Since the request may take longer than the budget, this number
	// can get negative, and is summed across requests.
	t.sleepFor += t.perRequest - now.Sub(t.last)

	// We shouldn't allow sleepFor to get too negative, since it would mean that
	// a service that slowed down a lot for a short period of time would get
	// a much higher RPS following that.
	if t.sleepFor < t.maxSlack {
		t.sleepFor = t.maxSlack
	}

	// If sleepFor is positive, then we should sleep now.
	if t.sleepFor > 0 {
		t.clock.Sleep(t.sleepFor)
		t.last = now.Add(t.sleepFor)
		t.sleepFor = 0
	} else {
		t.last = now
	}

	return t.last
}
```

第二版本
```go
// Take blocks to ensure that the time spent between multiple
// Take calls is on average time.Second/rate.
func (t *limiter) Take() time.Time {
	newState := state{}
	taken := false
	for !taken {
		now := t.clock.Now()

		previousStatePointer := atomic.LoadPointer(&t.state)
		oldState := (*state)(previousStatePointer)

		newState = state{}
		newState.last = now

		// If this is our first request, then we allow it.
		if oldState.last.IsZero() {
			taken = atomic.CompareAndSwapPointer(&t.state, previousStatePointer, unsafe.Pointer(&newState))
			continue
		}

		// sleepFor calculates how much time we should sleep based on
		// the perRequest budget and how long the last request took.
		// Since the request may take longer than the budget, this number
		// can get negative, and is summed across requests.
		newState.sleepFor += t.perRequest - now.Sub(oldState.last)
		// We shouldn't allow sleepFor to get too negative, since it would mean that
		// a service that slowed down a lot for a short period of time would get
		// a much higher RPS following that.
		if newState.sleepFor < t.maxSlack {
			newState.sleepFor = t.maxSlack
		}
		if newState.sleepFor > 0 {
			newState.last = newState.last.Add(newState.sleepFor)
		}
		taken = atomic.CompareAndSwapPointer(&t.state, previousStatePointer, unsafe.Pointer(&newState))
	}
	t.clock.Sleep(newState.sleepFor)
	return newState.last
}
```

第二版本采用原子操作+for的自旋操作来替代lock操作，这样做的目的是减少协程锁竞争。 两个版本不管是用锁还是原子操作本质都是让请求排队，第一版本存在锁竞争，然后排队sleep，第二版本避免锁竞争，但是所有协程可能很快跳出for循环然后都会在sleep处sleep。
