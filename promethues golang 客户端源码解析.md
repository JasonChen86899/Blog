Promethues Golang 客户端 源码解析

监控利器 promethues，线上产品必备的监控组件，话不多说，开始client端源码旅程
分为三个部分
1. Register
2. Collector
3. Push Gateway client

### 概念简介
Desc：表示一个监控的指标，客户端代码基于它唯一标记一个采集指标
promethues.Metric: 这是一个interface 接口，定义一个指标需要实现两个方法：
	Desc() 和 Write(&dto.Metric)
```go
type Metric interface {
	// Desc returns the descriptor for the Metric. This method idempotently
	// returns the same descriptor throughout the lifetime of the
	// Metric. The returned descriptor is immutable by contract. A Metric
	// unable to describe itself must return an invalid descriptor (created
	// with NewInvalidDesc).
	Desc() *Desc
	// Write encodes the Metric into a "Metric" Protocol Buffer data
	// transmission object.
	//
	// Metric implementations must observe concurrency safety as reads of
	// this metric may occur at any time, and any blocking occurs at the
	// expense of total performance of rendering all registered
	// metrics. Ideally, Metric implementations should support concurrent
	// readers.
	//
	// While populating dto.Metric, it is the responsibility of the
	// implementation to ensure validity of the Metric protobuf (like valid
	// UTF-8 strings or syntactically valid metric and label names). It is
	// recommended to sort labels lexicographically. Callers of Write should
	// still make sure of sorting if they depend on it.
	Write(*dto.Metric) error
	// TODO(beorn7): The original rationale of passing in a pre-allocated
	// dto.Metric protobuf to save allocations has disappeared. The
	// signature of this method should be changed to "Write() (*dto.Metric,
	// error)".
}
```
dto.Metric：定义在pb 文件中 表示一个传送给 promethues server 端的 采集到的指标值，可以理解为 Desc 的一个实例值
labelPairs：定义在pb文件中 在Metric 结构体内，表示一个label key-value 键值对 实例值

### Collector 源码解析
采集器 顾名思义 我们所关心的所需要监控的指标的采集器，比如 go_collector 这是采集go 环境指标的采集，采集 在 高并发场景下我们所关心的指标：goroutine 的数量，os thread 的数量，gc 时长。
源码剖析：
```go
// Collector的接口
type Collector interface {
	Describe(chan<- *Desc)
	Collect(chan<- Metric)
}
```


```go
Describe(chan<- *Desc)
```
用来创建 Desc，一个采集器有很多 Desc 也就是一个采集器可以收集很多指标
```go
Collect(chan<- Metric)
```
用来收集指标值，这是采集指标的采集入口，由Register Gather 方法进行调用 后续在Register 源码讲解中会涉及

收集方法的传参是一个只写channel，意思就是由采集器将采集到的数值写入 channel。
传入的值是Metric 的一个实例值，Metric类型有四种：counter， gauge，histogram，summary。

从源码设计的角度开看 collector 接口的具体实现主要有 metricVec，它有四个主要的子结构体：
CounterVec，GaugeVec，HistogramVec，SummaryVec。从原作者的注释来看，其意思是带有promethues.Metric 的 collector。
从笔者对源码的理解程度来看，这样设计的原因：将Register接口对具有单一指标的collector的注册和对拥有多个指标的collector的注册统一到一起。
从业务角度来看，单一指标占大多数，也是经常使用的。多个指标collector却往往是单独提供的特殊collector 比如，go_collector ，这是上报go 运行环境指标的采集器，其内部有很多个采集指标，里面的逻辑则是包含特殊处理的value Metric(这也是一个Metric)

### Register源码解析
注册
```go
type Registerer interface {
	// Register registers a new Collector to be included in metrics
	// collection. It returns an error if the descriptors provided by the
	// Collector are invalid or if they — in combination with descriptors of
	// already registered Collectors — do not fulfill the consistency and
	// uniqueness criteria described in the documentation of metric.Desc.
	//
	// If the provided Collector is equal to a Collector already registered
	// (which includes the case of re-registering the same Collector), the
	// returned error is an instance of AlreadyRegisteredError, which
	// contains the previously registered Collector.
	//
	// A Collector whose Describe method does not yield any Desc is treated
	// as unchecked. Registration will always succeed. No check for
	// re-registering (see previous paragraph) is performed. Thus, the
	// caller is responsible for not double-registering the same unchecked
	// Collector, and for providing a Collector that will not cause
	// inconsistent metrics on collection. (This would lead to scrape
	// errors.)
	Register(Collector) error
	// MustRegister works like Register but registers any number of
	// Collectors and panics upon the first registration that causes an
	// error.
	MustRegister(...Collector)
	// Unregister unregisters the Collector that equals the Collector passed
	// in as an argument.  (Two Collectors are considered equal if their
	// Describe method yields the same set of descriptors.) The function
	// returns whether a Collector was unregistered. Note that an unchecked
	// Collector cannot be unregistered (as its Describe method does not
	// yield any descriptor).
	//
	// Note that even after unregistering, it will not be possible to
	// register a new Collector that is inconsistent with the unregistered
	// Collector, e.g. a Collector collecting metrics with the same name but
	// a different help string. The rationale here is that the same registry
	// instance must only collect consistent metrics throughout its
	// lifetime.
	Unregister(Collector) bool
}
```

Register实现了Registerer的接口，接口里面的三个方法的主要作用很明显，下面详细讲解下细节，话不多说，开启代码详解
```go
type Registry struct {
	mtx                   sync.RWMutex
	collectorsByID        map[uint64]Collector // ID is a hash of the descIDs.
	descIDs               map[uint64]struct{}
	dimHashesByName       map[string]uint64
	uncheckedCollectors   []Collector
	pedanticChecksEnabled bool
}

// Register implements Registerer.
func (r *Registry) Register(c Collector) error {
	var (
		descChan           = make(chan *Desc, capDescChan)
		newDescIDs         = map[uint64]struct{}{}
		newDimHashesByName = map[string]uint64{}
		collectorID        uint64 // Just a sum of all desc IDs.
		duplicateDescErr   error
	)
	go func() {
		// 调用采集器的 Describe进行collector 里面Desc的收集
		c.Describe(descChan)
		close(descChan)
	}()
	r.mtx.Lock()
	defer func() {
		// Drain channel in case of premature return to not leak a goroutine.
		防止协程泄露，这里需要重点讲解下, 因为descChan的默认长度只有10，所以当
Collector的desc 比较多，下面从channel取数据的程序提前return，那么 上面的goroutine将会阻塞，高并发的情况下会出现协程泄漏
		for range descChan {
		}
		r.mtx.Unlock()
	}()
	// Conduct various tests...
	for desc := range descChan {

		// Is the descriptor valid at all?
		if desc.err != nil {
			return fmt.Errorf("descriptor %s is invalid: %s", desc, desc.err)
		}

		// Is the descID unique?
		// (In other words: Is the fqName + constLabel combination unique?)
		if _, exists := r.descIDs[desc.id]; exists {
			duplicateDescErr = fmt.Errorf("descriptor %s already exists with the same fully-qualified name and const label values", desc)
		}
		// If it is not a duplicate desc in this collector, add it to
		// the collectorID.  (We allow duplicate descs within the same
		// collector, but their existence must be a no-op.)
		if _, exists := newDescIDs[desc.id]; !exists {
			newDescIDs[desc.id] = struct{}{}
			collectorID += desc.id
		}

		// Are all the label names and the help string consistent with
		// previous descriptors of the same name?
		// First check existing descriptors...
		if dimHash, exists := r.dimHashesByName[desc.fqName]; exists {
			if dimHash != desc.dimHash {
				return fmt.Errorf("a previously registered descriptor with the same fully-qualified name as %s has different label names or a different help string", desc)
			}
		} else {
			// ...then check the new descriptors already seen.
			if dimHash, exists := newDimHashesByName[desc.fqName]; exists {
				if dimHash != desc.dimHash {
					return fmt.Errorf("descriptors reported by collector have inconsistent label names or help strings for the same fully-qualified name, offender is %s", desc)
				}
			} else {
				newDimHashesByName[desc.fqName] = desc.dimHash
			}
		}
	}
	// A Collector yielding no Desc at all is considered unchecked.
	if len(newDescIDs) == 0 {
		r.uncheckedCollectors = append(r.uncheckedCollectors, c)
		return nil
	}
	if existing, exists := r.collectorsByID[collectorID]; exists {
		return AlreadyRegisteredError{
			ExistingCollector: existing,
			NewCollector:      c,
		}
	}
	// If the collectorID is new, but at least one of the descs existed
	// before, we are in trouble.
	if duplicateDescErr != nil {
		return duplicateDescErr
	}

	// Only after all tests have passed, actually register.
	r.collectorsByID[collectorID] = c
	for hash := range newDescIDs {
		r.descIDs[hash] = struct{}{}
	}
	for name, dimHash := range newDimHashesByName {
		r.dimHashesByName[name] = dimHash
	}
	return nil
}
```

由上述代码可以看出注册只是进行了Desc 指标的记录，那么如何进行指标值的采集与记录，如何上报。笔者带着问题进行了相关源码的阅读

采集
```go
// Gatherer is the interface for the part of a registry in charge of gathering
// the collected metrics into a number of MetricFamilies. The Gatherer interface
// comes with the same general implication as described for the Registerer
// interface.
type Gatherer interface {
	// Gather calls the Collect method of the registered Collectors and then
	// gathers the collected metrics into a lexicographically sorted slice
	// of uniquely named MetricFamily protobufs. Gather ensures that the
	// returned slice is valid and self-consistent so that it can be used
	// for valid exposition. As an exception to the strict consistency
	// requirements described for metric.Desc, Gather will tolerate
	// different sets of label names for metrics of the same metric family.
	//
	// Even if an error occurs, Gather attempts to gather as many metrics as
	// possible. Hence, if a non-nil error is returned, the returned
	// MetricFamily slice could be nil (in case of a fatal error that
	// prevented any meaningful metric collection) or contain a number of
	// MetricFamily protobufs, some of which might be incomplete, and some
	// might be missing altogether. The returned error (which might be a
	// MultiError) explains the details. Note that this is mostly useful for
	// debugging purposes. If the gathered protobufs are to be used for
	// exposition in actual monitoring, it is almost always better to not
	// expose an incomplete result and instead disregard the returned
	// MetricFamily protobufs in case the returned error is non-nil.
	Gather() ([]*dto.MetricFamily, error)
}
```

### Gather
Register 不仅实现了Registerer接口还实现了Gatherer接口，这个接口主要是用来采集具体的指标值。具体看下里面的实现
```go
func (r *Registry) Gather() ([]*dto.MetricFamily, error) {
	var (
		checkedMetricChan   = make(chan Metric, capMetricChan)
		uncheckedMetricChan = make(chan Metric, capMetricChan)
		metricHashes        = map[uint64]struct{}{}
		wg                  sync.WaitGroup
		errs                MultiError          // The collected errors to return in the end.
		registeredDescIDs   map[uint64]struct{} // Only used for pedantic checks
	)

	r.mtx.RLock()
	goroutineBudget := len(r.collectorsByID) + len(r.uncheckedCollectors)
处理指标Metric后存放结果的切片
metricFamiliesByName := make(map[string]*dto.MetricFamily, len(r.dimHashesByName))
	checkedCollectors := make(chan Collector, len(r.collectorsByID))
	uncheckedCollectors := make(chan Collector, len(r.uncheckedCollectors))
	for _, collector := range r.collectorsByID {
		checkedCollectors <- collector
	}
	for _, collector := range r.uncheckedCollectors {
		uncheckedCollectors <- collector
	}
	// In case pedantic checks are enabled, we have to copy the map before
	// giving up the RLock.
	if r.pedanticChecksEnabled {
		registeredDescIDs = make(map[uint64]struct{}, len(r.descIDs))
		for id := range r.descIDs {
			registeredDescIDs[id] = struct{}{}
		}
	}
	r.mtx.RUnlock()

	wg.Add(goroutineBudget)

	collectWorker := func() {
		for {
			select {
			case collector := <-checkedCollectors:
调用clollector的Clollect方法
				collector.Collect(checkedMetricChan)
			case collector := <-uncheckedCollectors:
				collector.Collect(uncheckedMetricChan)
			default:
				return
			}
			wg.Done()
		}
	}

	// Start the first worker now to make sure at least one is running.
	go collectWorker()
	goroutineBudget--

	// Close checkedMetricChan and uncheckedMetricChan once all collectors
	// are collected.
	go func() {
		wg.Wait()
		close(checkedMetricChan)
		close(uncheckedMetricChan)
	}()

	// Drain checkedMetricChan and uncheckedMetricChan in case of premature return.
	defer func() {
		if checkedMetricChan != nil {
			for range checkedMetricChan {
			}
		}
		if uncheckedMetricChan != nil {
			for range uncheckedMetricChan {
			}
		}
	}()

	// Copy the channel references so we can nil them out later to remove
	// them from the select statements below.
针对收集到的指标值进行处理，处理过程：调用processMetric函数将promethues.Metric 转换放进 metricFamiliesByName 切片中
	cmc := checkedMetricChan
	umc := uncheckedMetricChan

	for {
		select {
		case metric, ok := <-cmc:
			if !ok {
				cmc = nil
				break
			}
			errs.Append(processMetric(
				metric, metricFamiliesByName,
				metricHashes,
				registeredDescIDs,
			))
		case metric, ok := <-umc:
			if !ok {
				umc = nil
				break
			}
			errs.Append(processMetric(
				metric, metricFamiliesByName,
				metricHashes,
				nil,
			))
		default:
			if goroutineBudget <= 0 || len(checkedCollectors)+len(uncheckedCollectors) == 0 {
				// All collectors are already being worked on or
				// we have already as many goroutines started as
				// there are collectors. Do the same as above,
				// just without the default.
				select {
				case metric, ok := <-cmc:
					if !ok {
						cmc = nil
						break
					}
					errs.Append(processMetric(
						metric, metricFamiliesByName,
						metricHashes,
						registeredDescIDs,
					))
				case metric, ok := <-umc:
					if !ok {
						umc = nil
						break
					}
					errs.Append(processMetric(
						metric, metricFamiliesByName,
						metricHashes,
						nil,
					))
				}
				break
			}
			// Start more workers.
			go collectWorker()
			goroutineBudget--
			runtime.Gosched()
		}
		// Once both checkedMetricChan and uncheckdMetricChan are closed
		// and drained, the contraption above will nil out cmc and umc,
		// and then we can leave the collect loop here.
		if cmc == nil && umc == nil {
			break
		}
	}
	return internal.NormalizeMetricFamilies(metricFamiliesByName), errs.MaybeUnwrap()
}
```

这里涉及到clollector的Collect 方法 和 如何 细节处理 Metric的
processMetric 方法，接下里笔者用两个小章节深入分析这两个方法
#### Collect
```go
// Collect implements Collector.
func (m *metricMap) Collect(ch chan<- Metric) {
	m.mtx.RLock()
	defer m.mtx.RUnlock()

	for _, metrics := range m.metrics {
		for _, metric := range metrics {
将指标值放入ch channel
			ch <- metric.metric
		}
	}
}
```
可能会很困惑这里的metricMap。metricMap是metricVec的子结构体，所以这个方法也是四个指标Vec结构体的Collect方法。

WithLabelValues(lvs ...string) 或者 With(labels Labels)
可以看到是metricMap结构体内部存在metrics数组切片。这个数组切片里的值的add 其实是通过 Vec的 WithLabelValues(lvs ...string) 或者 With(labels Labels) 方法进行，在Vec 四个指标结构体里面的 都具有相同的方法，其本质是获取metricMap里面 metrics 然后将里面的值进行相应的操作，比如 counter 的 Add Inc等方法，gauge 的 Set 方法，histogram 和 summary 的 Observe 方法等
四个指标Vec 的 WithLabelValues(lvs ...string) 或者 With(labels Labels) 方法内部：
获取promethues.Metric指标，如果获取不到就创建一个metric
再根据前面的metric 进行相应的操作，前面有叙述，四个指标的操作

#### WithLabelValues：
```go
func (m *metricVec) getMetricWithLabelValues(lvs ...string) (Metric, error) {
	h, err := m.hashLabelValues(lvs)
	if err != nil {
		return nil, err
	}

	return m.metricMap.getOrCreateMetricWithLabelValues(h, lvs, m.curry), nil
}
```

#### With：
```go
func (m *metricVec) getMetricWith(labels Labels) (Metric, error) {
	h, err := m.hashLabels(labels)
	if err != nil {
		return nil, err
	}

	return m.metricMap.getOrCreateMetricWithLabels(h, labels, m.curry), nil
}
```
上述是深入源码后里 metricVec和metricMap的内部方法，具体操作是根据计算出得 hash值找到metric，也就是 metrics[h] 如果没有就进行创建
```go
func (m *metricMap) getOrCreateMetricWithLabelValues(
	hash uint64, lvs []string, curry []curriedLabelValue,
) Metric {
先用读锁get
	m.mtx.RLock()
	metric, ok := m.getMetricWithHashAndLabelValues(hash, lvs, curry)
	m.mtx.RUnlock()
	if ok {
		return metric
	}
如果需要create 用写锁，也是先get 然后再create 保证协程安全
	m.mtx.Lock()
	defer m.mtx.Unlock()
	metric, ok = m.getMetricWithHashAndLabelValues(hash, lvs, curry)
	if !ok {
		inlinedLVs := inlineLabelValues(lvs, curry)
新建一个metric，这里newMetric是个方法，这个方法由在创建四个指标时传入
比如 counter：
func NewCounterVec(opts CounterOpts, labelNames []string) *CounterVec {
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &CounterVec{
创建新的MetricVec，创建的时候传入匿名函数 func(lvs ...string) Metric
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			if len(lvs) != len(desc.variableLabels) {
				panic(makeInconsistentCardinalityError(desc.fqName, desc.variableLabels, lvs))
			}
创建counter 并且返回
			result := &counter{desc: desc, labelPairs: makeLabelPairs(desc, lvs)}
			result.init(result) // Init self-collection.
			return result
		}),
	}
}

		metric = m.newMetric(inlinedLVs...)
metricWithLabelValues是内部的一个结构体，其实就是metrics切片里面元素的本质
		m.metrics[hash] = append(m.metrics[hash], metricWithLabelValues{values: inlinedLVs, metric: metric})
	}
	return metric
}
```
返回metric后就可以进行相应的具体操作。
根据笔者的源码解读和思考，指标的变化其实本质就是Metric的手动变化，笔者之所以叫做手动是因为这些变化操作都需要业务方自己去调用。

### Push 源码解析
API接口
```go
func (p *Pusher) Push() error {
	return p.push("PUT")
}
```

add 顾名思义就是增加，业务场景基本用的是Add
```go
func (p *Pusher) Add() error {
	return p.push("POST")
}
```

push：删除原有的所有指标并推送新的指标，对应put方法

pushadd：更新已有的所有指标，对应post方法

delete：删除指标，对应delete方法

深入解析push方法
```go
func (p *Pusher) push(method string) error {
	if p.error != nil {
		return p.error
	}
	urlComponents := []string{url.QueryEscape(p.job)}
	for ln, lv := range p.grouping {
		urlComponents = append(urlComponents, ln, lv)
	}
	pushURL := fmt.Sprintf("%s/metrics/job/%s", p.url, strings.Join(urlComponents, "/"))
这里是调用gathers的Gather方法，内部遍历每个gather的Gather方法，Gather方法的讲解在上文中
	mfs, err := p.gatherers.Gather()
	if err != nil {
		return err
	}
	buf := &bytes.Buffer{}
	enc := expfmt.NewEncoder(buf, p.expfmt)
	// Check for pre-existing grouping labels:
	for _, mf := range mfs {
		for _, m := range mf.GetMetric() {
			for _, l := range m.GetLabel() {
				if l.GetName() == "job" {
					return fmt.Errorf("pushed metric %s (%s) already contains a job label", mf.GetName(), m)
				}
				if _, ok := p.grouping[l.GetName()]; ok {
					return fmt.Errorf(
						"pushed metric %s (%s) already contains grouping label %s",
						mf.GetName(), m, l.GetName(),
					)
				}
			}
		}
		enc.Encode(mf)
	}
这里发送http请求
	req, err := http.NewRequest(method, pushURL, buf)
	if err != nil {
		return err
	}
	if p.useBasicAuth {
		req.SetBasicAuth(p.username, p.password)
	}
	req.Header.Set(contentTypeHeader, string(p.expfmt))
	resp, err := p.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode != 202 {
		body, _ := ioutil.ReadAll(resp.Body) // Ignore any further error as this is for an error message only.
		return fmt.Errorf("unexpected status code %d while pushing to %s: %s", resp.StatusCode, pushURL, body)
	}
	return nil
}
```

### Summary和Histogram 算法分析
这是笔者附加的一个章节，这里涉及一个基于流的计算有偏差的分位数值的算法，这个算法的论文地址：
http://www.cs.rutgers.edu/~muthu/bquant.pdf

首先介绍下什么是分位数，分位数是统计学看里面概念，在统计学中经常被提及，下面笔者通过一个例子来简单解释一下。

假设有一千名学生参加了某次考试，
学生A得了75分，排名603，603/1000＝60.3%
学生B得了94分，排名28，28/1000=2.8%
此时，A大约在60.3%的位置上，而B大约在2.8%的位置上。即在60.3%的位置上约75分, 2.8%的位置上约94分。

对应四分位数的就很好解释了，分别在25%, 50%, 75%位置上的数。假设考生甲乙丙丁考试成绩分别为80,71,61，对应的名次分别为250,500,750名，那么对应的四分位数分别就为80,71,61。

beorn7/perks/quantile 代码库基于上述论文进行了工程实现，具体算法笔者大致浏览了下，主要分为3个主要函数：
* merge
* compress
* query

其中merge和compress一起使用，先merge 样本再compress样本，query主要用来查询具体分位数值，里面实现的逻辑需要结合论文去理解，笔者暂时没有研究论文提出的算法以及理论依据。
