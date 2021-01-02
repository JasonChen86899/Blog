Go-micro 框架解析

### Server
Server包下的集中了server端代码的实现
两个rpc协议：
```go
// Server is a simple micro server abstraction
type Server interface {
	Options() Options
	Init(...Option) error
	Handle(Handler) error
	NewHandler(interface{}, ...HandlerOption) Handler
	NewSubscriber(string, interface{}, ...SubscriberOption) Subscriber
	Subscribe(Subscriber) error
	Start() error
	Stop() error
	String() string
}
```

#### Mucp协议
rpc_server 实现上面的接口。
这是go micro原生的协议，transport包是其主要的依赖包 包下有对底层传输协议的封装。底层的协议主要是http grpc quic。
Mucp 协议的数据格式很简单：
```go
type Message struct {
	Header map[string]string
	Body   []byte
}
```
#### grpc协议
我们在上面的协议中也看到了grpc 但是这里的grpc跟上述的grpc是完全不一致的。简单来说这里的grpc是和rpc 一个级别的。都是实现server的接口
数据协议采用pb自定义协议。
在microKit项目中 采用此方式进行。
为了进行go-micro 注册框架的适配 注册方式跟传统grpc 注册是有区别的。注册都是向zk 等远程注册中心进行的注册。



### Client

#### Registry
服务注册有些地方也叫服务治理其实本质都是一样的：将服务的ip address 和 port 注册到分布式协调组件中（协调组件有zk，etcd等）。go-micro对注册服务的格式进行了统一，如下：
分布式注册格式如下
```go
type Node struct {
	Id       string            `json:"id"` // 注册节点 唯一性 ID 框架采用address（ip+port）
	Address  string            `json:"address"`// 注册 节点的 ip address:port
	Metadata map[string]string `json:"metadata"`// 注册节点的 metadata值
}

type Service struct {
	Name      string            `json:"name"`// 服务名 框架采用%s-%s-%s格式  gamerp-play-test-id
	Version   string            `json:"version"` // 版本号 默认采用时间戳
	Metadata  map[string]string `json:"metadata"` // metadata的值
	Endpoints []*Endpoint       `json:"endpoints"` //注册的时候只会进行赋值 endpoint 注册的方法和注册的订阅者，但实际写注册中心是没用的 不会用到的
	Nodes     []*Node           `json:"nodes"` // 注册的节点信息
}
```

Go-micro的设计是不绑定于特定的注册中心和rpc通信，抽象Node Service 的结构体

### Grpc-go-micro

#### Client
#### Server
#### Registry
不同于正常的服务注册 这里的注册指的是服务类中方法在本地的注册，这是一个比较重要的部分。任何rpc都需要进行这步操作 接收到请求后找到相应的调用方法然后进行反射调用。

注册的方式跟grpc源码注册方式是相似的。
反射获取结构体方法
针对各个方法进行check
```go
type service struct {
	name   string                 // name of service
	rcvr   reflect.Value          // receiver of methods for the service
	typ    reflect.Type           // type of the receiver
	method map[string]*methodType // registered methods
}
采用如上结构体对需要注册的服务结构体进行封装处理

type methodType struct {
	method      reflect.Method
	ArgType     reflect.Type
	ReplyType   reflect.Type
	ContextType reflect.Type
	stream      bool
}

采用如上结构体对需要注册的服务结构体的每个方法进行封装处理

最后采用map方式存储如下：server.serviceMap[s.name] = s
```
#### Handler
这里所指的其实是grpc 的 handler，
