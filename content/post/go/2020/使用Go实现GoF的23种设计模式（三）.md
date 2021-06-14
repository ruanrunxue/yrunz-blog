---
title: 使用Go实现GoF的23种设计模式（三）
date: 2020-09-06
categories:
 - Golang
tags:
 - golang
 - 设计模式
---

## 前言

上一篇文章[《使用Go实现GoF的23种设计模式（二）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（二）)中，我们介绍了**结构型模式**（Structural Pattern）中的组合模式、适配器模式和桥接模式。本文将会介绍完剩下的几种结构型模式，代理模式、装饰模式、外观模式和享元模式。本文将会继续采用消息处理系统作为例子，如果对该例子不清楚，请移步[《使用Go实现GoF的23种设计模式（一）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（一）)和[《使用Go实现GoF的23种设计模式（二）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（二）)对其相关的设计和实现进行了解。

## 代理模式（Proxy Pattern）

![代理模式结构](https://tva1.sinaimg.cn/large/007S8ZIlly1gi7cqybscgj31am0kcu0x.jpg)

### 简介

**代理模式为一个对象提供一种代理以控制对该对象的访问**，它是一个使用率非常高的设计模式，即使在现实生活中，也是很常见，比如演唱会门票黄牛。假设你需要看一场演唱会，但是官网上门票已经售罄，于是就当天到现场通过黄牛高价买了一张。在这个例子中，黄牛就相当于演唱会门票的代理，在正式渠道无法购买门票的情况下，你通过代理完成了该目标。

从演唱会门票的例子我们也可以看出，使用代理模式的关键在于**当Client不方便直接访问一个对象时，提供一个代理对象控制该对象的访问**。Client实际上访问的是代理对象，代理对象会将Client的请求转给本体对象去处理。

在程序设计中，代理模式也分为好几种：

1、**远程代理**（remote proxy），远程代理适用于提供服务的对象处在远程的机器上，通过普通的函数调用无法使用服务，需要经过远程代理来完成。*因为并不能直接访问本体对象，所有远程代理对象通常不会直接持有本体对象的引用，而是持有远端机器的地址，通过网络协议去访问本体对象*。

2、**虚拟代理**（virtual proxy），在程序设计中常常会有一些重量级的服务对象，如果一直持有该对象实例会非常消耗系统资源，这时可以通过虚拟代理来对该对象进行延迟初始化。

3、**保护代理**（protection proxy），保护代理用于控制对本体对象的访问，常用于需要给Client的访问加上权限验证的场景。

4、**缓存代理**（cache proxy），缓存代理主要在Client与本体对象之间加上一层缓存，用于加速本体对象的访问，常见于连接数据库的场景。

5、**智能引用**（smart reference），智能引用为本体对象的访问提供了额外的动作，常见的实现为C++中的智能指针，为对象的访问提供了计数功能，当访问对象的计数为0时销毁该对象。

这几种代理都是一样的实现原理，下面我们将介绍远程代理的Go语言实现。

### Go实现

考虑要将消息处理系统输出到数据存储到一个数据库中，数据库的接口如下：

```go
package db
...
// Key-Value数据库接口
type KvDb interface {
	// 存储数据
	// 其中reply为操作结果，存储成功为true，否则为false
	// 当连接数据库失败时返回error，成功则返回nil
	Save(record Record, reply *bool) error
	// 根据key获取value，其中value通过函数参数中指针类型返回
	// 当连接数据库失败时返回error，成功则返回nil
	Get(key string, value *string) error
}

type Record struct {
	Key   string
	Value string
}
```

数据库是一个Key-Value数据库，使用`map`存储数据，下面为数据库的服务端实现，`db.Server`实现了`db.KvDb`接口：

```go
package db
...
// 数据库服务端实现
type Server struct {
	// 采用map存储key-value数据
	data map[string]string
}

func (s *Server) Save(record Record, reply *bool) error {
	if s.data == nil{
		s.data = make(map[string]string)
	}
	s.data[record.Key] = record.Value
	*reply = true
	return nil
}

func (s *Server) Get(key string, reply *string) error {
	val, ok := s.data[key]
	if !ok {
		*reply = ""
		return errors.New("Db has no key " + key)
	}
	*reply = val
	return nil
}
```

消息处理系统和数据库并不在同一台机器上，因此消息处理系统不能直接调用`db.Server`的方法进行数据存储，像这种服务提供者和服务使用者不在同一机器上的场景，使用远程代理再适合不过了。

远程代理中，最常见的一种实现是**远程过程调用**（Remote Procedure Call，简称 **RPC**），它允许客户端应用可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法。在Go语言领域，除了大名鼎鼎的**gRPC**，Go标准库`net/rpc`包里也提供了RPC的实现。下面，我们通过`net/rpc`对外提供数据库服务端的能力：

```go
package db
...
// 启动数据库，对外提供RPC接口进行数据库的访问
func Start() {
	rpcServer := rpc.NewServer()
	server := &Server{data: make(map[string]string)}
  // 将数据库接口注册到RPC服务器上
	if err := rpcServer.Register(server); err != nil {
		fmt.Printf("Register Server to rpc failed, error: %v", err)
		return
	}
	l, err := net.Listen("tcp", "127.0.0.1:1234")
	if err != nil {
		fmt.Printf("Listen tcp failed, error: %v", err)
		return
	}
	go rpcServer.Accept(l)
	time.Sleep(1 * time.Second)
	fmt.Println("Rpc server start success.")
}
```

到目前为止，我们已经为数据库提供了对外访问的方式。现在，我们需要一个远程代理来连接数据库服务端，并进行相关的数据库操作。对消息处理系统而言，它不需要，也不应该知道远程代理与数据库服务端交互的底层细节，这样可以减轻系统之间的耦合。因此，远程代理需要实现`db.KvDb`：

```go
package db
...
// 数据库服务端远程代理，实现db.KvDb接口
type Client struct {
	// RPC客户端
	cli *rpc.Client
}

func (c *Client) Save(record Record, reply *bool) error {
	var ret bool
	// 通过RPC调用服务端的接口
	err := c.cli.Call("Server.Save", record, &ret)
	if err != nil {
		fmt.Printf("Call db Server.Save rpc failed, error: %v", err)
		*reply = false
		return err
	}
	*reply = ret
	return nil
}

func (c *Client) Get(key string, reply *string) error {
	var ret string
	// 通过RPC调用服务端的接口
	err := c.cli.Call("Server.Get", key, &ret)
	if err != nil {
		fmt.Printf("Call db Server.Get rpc failed, error: %v", err)
		*reply = ""
		return err
	}
	*reply = ret
	return nil
}

// 工厂方法，返回远程代理实例
func CreateClient() *Client {
	rpcCli, err := rpc.Dial("tcp", "127.0.0.1:1234")
	if err != nil {
		fmt.Printf("Create rpc client failed, error: %v.", err)
		return nil
	}
	return &Client{cli: rpcCli}
}
```

作为远程代理的`db.Client`并没有直接持有`db.Server`的引用，而是持有了它的`ip:port`，通过RPC客户端调用了它的方法。 

![数据库远程代理结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gig7p19gkdj318m0k4qv5.jpg)

接下来，我们需要为消息处理系统实现一个新的`Output`插件`DbOutput`，调用`db.Client`远程代理，将消息存储到数据库上。

> 在[《使用Go实现GoF的23种设计模式（二）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（二）)中我们为`Plugin`引入生命周期的三个方法`Start`、`Stop`、`Status`之后，每新增一个新的插件，都需要实现这三个方法。但是大多数插件的这三个方法的逻辑基本一致，因此导致了一定程度的代码冗余。对于重复代码问题，有什么好的解决方法呢？**组合模式**！
>
> 下面，我们使用组合模式将这个方法提取成一个新的对象`LifeCycle`，这样新增一个插件时，只需将`LifeCycle`作为匿名成员（**嵌入组合**），就能解决冗余代码问题了。
>
> ```go
> package plugin
> ...
> type LifeCycle struct {
> 	name   string
> 	status Status
> }
> 
> func (l *LifeCycle) Start() {
> 	l.status = Started
> 	fmt.Printf("%s plugin started.\n", l.name)
> }
> 
> func (l *LifeCycle) Stop() {
> 	l.status = Stopped
> 	fmt.Printf("%s plugin stopped.\n", l.name)
> }
> 
> func (l *LifeCycle) Status() Status {
> 	return l.status
> }
> ```

`DbOutput`的实现如下，它持有一个远程代理，通过后者将消息存储到远端的数据库中。

```go
package plugin
...
type DbOutput struct {
	LifeCycle
	// 操作数据库的远程代理
	proxy db.KvDb
}

func (d *DbOutput) Send(msg *msg.Message) {
	if d.status != Started {
		fmt.Printf("%s is not running, output nothing.\n", d.name)
		return
	}
	record := db.Record{
		Key:   "db",
		Value: msg.Body.Items[0],
	}
	reply := false
	err := d.proxy.Save(record, &reply)
	if err != nil || !reply {
		fmt.Println("Save msg to db server failed.")
	}
}

func (d *DbOutput) Init() {
	d.proxy = db.CreateClient()
	d.name = "db output"
}
```

测试代码如下：

```go
package test
...
func TestDbOutput(t *testing.T) {
	db.Start()
	config := pipeline.Config{
		Name: "pipeline3",
		Input: plugin.Config{
			PluginType: plugin.InputType,
			Name:       "hello",
		},
		Filter: plugin.Config{
			PluginType: plugin.FilterType,
			Name:       "upper",
		},
		Output: plugin.Config{
			PluginType: plugin.OutputType,
			Name:       "db",
		},
	}
	p := pipeline.Of(config)
	p.Start()
	p.Exec()

	// 验证DbOutput存储的正确性
	cli := db.CreateClient()
	var val string
	err := cli.Get("db", &val)
	if err != nil {
		t.Errorf("Get db failed, error: %v\n.", err)
	}
	if val != "HELLO WORLD" {
		t.Errorf("expect HELLO WORLD, but actual %s.", val)
	}
}
// 运行结果
=== RUN   TestDbOutput
Rpc server start success.
db output plugin started.
upper filter plugin started.
hello input plugin started.
Pipeline started.
--- PASS: TestDbOutput (1.01s)
PASS
```

## 装饰模式（Decorator Pattern）

![装饰模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gigmr38eysj31c40kqx6p.jpg)

### 简介

在程序设计中，我们常常需要为对象添加新的行为，很多同学的第一个想法就是扩展本体对象，通过继承的方式达到目的。但是使用继承不可避免地有如下两个弊端：（1）继承时静态的，在编译期间就已经确定，无法在运行时改变对象的行为。（2）子类只能有一个父类，当需要添加的新功能太多时，容易导致类的数量剧增。

对于这种场景，我们通常会使用**装饰模式**（Decorator Pattern）来解决，**它使用组合而非继承的方式，能够动态地为本体对象叠加新的行为**。理论上，只要没有限制，它可以一直把功能叠加下去。装饰模式最经典的应用当属Java的I/O流体系，通过装饰模式，使用者可以动态地为原始的输入输出流添加功能，比如按照字符串输入输出，添加缓存等，使得整个I/O流体系具有很高的可扩展性和灵活性。

从结构上看，装饰模式和代理模式具有很高的相似性，但是两种所强调的点不一样。**前者强调的是为本体对象添加新的功能，后者强调的是对本体对象的访问控制**。当然，代理模式中的智能引用在笔者看来就跟装饰模式完全一样了。

### Go实现

考虑为消息处理系统增加这样的一个功能，统计每个消息输入源分别产生了多少条消息，也就是分别统计每个`Input`产生`Message`的数量。最简单的方法是在每一个`Input`的`Receive`方法中进行打点统计，但是这样会导致统计代码与业务代码的耦合。如果统计逻辑发生了变化，就会产生**霰弹式修改**，随着`Input`类型的增多，相关代码也会变得越来越难维护。

更好的方法是将统计逻辑放到一个地方，并在每次调用`Input`的`Receive`方法后进行打点统计。而这恰好适合采用装饰模式，为`Input`（**本体对象**）提供打点统计功能（**新的行为**）。我们可以设计一个`InputMetricDecorator`作为`Input`的装饰器，在装饰器中完成打点统计的逻辑。

首先，我们需要设计一个用于统计每个`Input`产生`Message`数量的对象，该对象应该是一个全局唯一的，因此采用单例模式进行了实现：

```go
package metric
...
// 消息输入源统计，设计为单例
type input struct {
	// 存放统计结果，key为Input类型如hello、kafka
	// value为对应Input的消息统计
	metrics map[string]uint64
	// 统计打点时加锁
	mu      *sync.Mutex
}

// 给名称为inputName的Input消息计数加1
func (i *input) Inc(inputName string) {
	i.mu.Lock()
	defer i.mu.Unlock()
	if _, ok := i.metrics[inputName]; !ok {
		i.metrics[inputName] = 0
	}
	i.metrics[inputName] = i.metrics[inputName] + 1
}

// 输出当前所有打点的情况
func (i *input) Show() {
	fmt.Printf("Input metric: %v\n", i.metrics)
}

// 单例
var inputInstance = &input{
	metrics: make(map[string]uint64),
	mu:      &sync.Mutex{},
}

func Input() *input {
	return inputInstance
}
```

接下来我们开始实现`InputMetricDecorator`，它实现了`Input`接口，并持有一个本体对象`Input`。在`InputMetricDecorator`在`Receive`方法中调用本体`Input`的`Receive`方法，并完成统计动作。

```go
package plugin
...
type InputMetricDecorator struct {
	input Input
}

func (i *InputMetricDecorator) Receive() *msg.Message {
	// 调用本体对象的Receive方法
	record := i.input.Receive()
	// 完成统计逻辑
	if inputName, ok := record.Header.Items["input"]; ok {
		metric.Input().Inc(inputName)
	}
	return record
}

func (i *InputMetricDecorator) Start() {
	i.input.Start()
}

func (i *InputMetricDecorator) Stop() {
	i.input.Stop()
}

func (i *InputMetricDecorator) Status() Status {
	return i.input.Status()
}

func (i *InputMetricDecorator) Init() {
	i.input.Init()
}

// 工厂方法, 完成装饰器的创建
func CreateInputMetricDecorator(input Input) *InputMetricDecorator {
	return &InputMetricDecorator{input: input}
}
```

最后，我们在`Pipeline`的工厂方法上，为本体Input加上`InputMetricDecorator`代理：

```go
package pipeline
...
// 根据配置创建一个Pipeline实例
func Of(conf Config) *Pipeline {
	p := &Pipeline{}
	p.input = factoryOf(plugin.InputType).Create(conf.Input).(plugin.Input)
	p.filter = factoryOf(plugin.FilterType).Create(conf.Filter).(plugin.Filter)
	p.output = factoryOf(plugin.OutputType).Create(conf.Output).(plugin.Output)
	// 为本体Input加上InputMetricDecorator装饰器
	p.input = plugin.CreateInputMetricDecorator(p.input)
	return p
}
```

测试代码如下：

```go
package test
...
func TestInputMetricDecorator(t *testing.T) {
	p1 := pipeline.Of(pipeline.HelloConfig())
	p2 := pipeline.Of(pipeline.KafkaInputConfig())
	p1.Start()
	p2.Start()
	p1.Exec()
	p2.Exec()
	p1.Exec()

	metric.Input().Show()
}
// 运行结果
=== RUN   TestInputMetricDecorator
Console output plugin started.
Upper filter plugin started.
Hello input plugin started.
Pipeline started.
Console output plugin started.
Upper filter plugin started.
Kafka input plugin started.
Pipeline started.
Output:
	Header:map[content:text input:hello], Body:[HELLO WORLD]
Output:
	Header:map[content:text input:kafka], Body:[I AM MOCK CONSUMER.]
Output:
	Header:map[content:text input:hello], Body:[HELLO WORLD]
Input metric: map[hello:2 kafka:1]
--- PASS: TestInputMetricProxy (0.00s)
PASS
```

## 外观模式（Facade Pattern）

![外观模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gigourw3ksj316g0jgkjl.jpg)

### 简介

从结构上看，外观模式非常的简单，它主要是**为子系统提供了一个更高层次的对外统一接口，使得Client能够更友好地使用子系统的功能**。图中，Subsystem Class是子系统中对象的简称，它可能是一个对象，也可能是数十个对象的集合。外观模式降低了Client与Subsystem之间的耦合，只要Facade不变，不管Subsystem怎么变化，对于Client而言都是无感知的。

![使用外观模式进行系统优化](https://tva1.sinaimg.cn/large/007S8ZIlgy1gigpit27haj31k80r44qr.jpg)

外观模式在程序设计中用的非常多，比如我们在商城上点击`购买`的按钮，对于购买者而言，只看到了`购买`这一统一的接口，但是对于商城系统而言，其内部则进行了一系列的业务处理，比如库存检查、订单处理、支付、物流等等。外观模式极大地提升了用户体验，将用户从复杂的业务流程中解放了出来。

外观模式经常运用于**分层架构**上，通常我们都会为分层架构中的每一个层级提供一个或多个统一对外的访问接口，这样就能让各个层级之间的耦合性更低，使得系统的架构更加合理。

### Go实现

外观模式实现起来也很简单，还是考虑前面的消息处理系统。在`Pipeline`中，每一条消息会依次经过**Input->Filter->Output**的处理，代码实现起来就是这样：

```go
p := pipeline.Of(config)
message := p.input.Receive()
message = p.filter.Process(message)
p.output.Send(message)
```

但是，对于`Pipeline`的使用者而言，他可能并不关心消息具体的处理流程，他只需知道消息已经经过`Pipeline`处理即可。因此，我们需要设计一个简单的对外接口：

```go
package pipeline
...
func (p *Pipeline) Exec() {
	msg := p.input.Receive()
	msg = p.filter.Process(msg)
	p.output.Send(msg)
}
```

这样，使用者只需简单地调用`Exec`方法，就能完成一次消息的处理，测试代码如下：

```go
package test
...
func TestPipeline(t *testing.T) {
	p := pipeline.Of(pipeline.HelloConfig())
	p.Start()
  // 调用Exec方法完成一次消息的处理
	p.Exec()
}
// 运行结果
=== RUN   TestPipeline
console output plugin started.
upper filter plugin started.
hello input plugin started.
Pipeline started.
Output:
	Header:map[content:text input:hello], Body:[HELLO WORLD]
--- PASS: TestPipeline (0.00s)
PASS
```

## 享元模式（Flyweight Pattern）

![享元模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gigral1b1cj318i0hihdt.jpg)

### 简介

在程序设计中，我们常常会碰到一些很重型的对象，它们通常拥有很多的成员属性，当系统中充斥着大量的这些对象时，系统的内存将会承受巨大的压力。此外，频繁的创建这些对象也极大地消耗了系统的CPU。很多时候，这些重型对象里，大部分的成员属性都是固定的，这种场景下， 可以使用**享元模式**进行优化，将其中固定不变的部分设计成共享对象（享元，flyweight），这样就能节省大量的系统内存和CPU。

**享元模式摒弃了在每个对象中保存所有数据的方式， 通过共享多个对象所共有的相同状态， 让你能在有限的内存容量中载入更多对象**。

当我们决定对一个重型对象采用享元模式进行优化时，首先需要将该重型对象的属性划分为两类，能够共享的和不能共享的。前者我们称为**内部状态**（intrinsic state），存储在享元中，不随享元所处上下文的变化而变化；后者称为**外部状态**（extrinsic state），它的值取决于享元所处的上下文，因此不能共享。比如，文章A和文章B都引用了图片A，由于文章A和文章B的文字内容是不一样的，因此文字就是外部状态，不能共享；但是它们所引用的图片A是一样的，属于内部状态，因此可以将图片A设计为一个享元

**工厂模式**通常都会和享元模式结对出现，享元工厂提供了唯一获取享元对象的接口，这样Client就感知不到享元是如何共享的，降低了模块的耦合性。享元模式和**单例模式**有些类似的地方，都是在系统中共享对象，但是单例模式更关心的是**对象在系统中仅仅创建一次**，而享元模式更关心的是**如何在多个对象中共享相同的状态**。

### Go实现

假设现在需要设计一个系统，用于记录NBA中的球员信息、球队信息以及比赛结果。

球队`Team`的数据结构定义如下：

```go
package nba
...
type TeamId uint8

const (
	Warrior TeamId = iota
	Laker
)

type Team struct {
	Id      TeamId    // 球队ID
	Name    string    // 球队名称
	Players []*Player // 球队中的球员
}
```

球员`Player`的数据结构定义如下：

```go
package nba
...
type Player struct {
	Name string // 球员名字
	Team TeamId // 球员所属球队ID
}
```

比赛结果`Match`的数据结构定义如下：

```go
package nba
...
type Match struct {
	Date         time.Time // 比赛时间
	LocalTeam    *Team     // 主场球队
	VisitorTeam  *Team     // 客场球队
	LocalScore   uint8     // 主场球队得分
	VisitorScore uint8     // 客场球队得分
}

func (m *Match) ShowResult() {
	fmt.Printf("%s VS %s - %d:%d\n", m.LocalTeam.Name, m.VisitorTeam.Name,
		m.LocalScore, m.VisitorScore)
}
```

NBA中的一场比赛由两个球队，主场球队和客场球队，完成比赛，对应着代码就是，一个`Match`实例会持有2个`Team`实例。目前，NBA总共由30支球队，按照每个赛季每个球队打82场常规赛算，一个赛季总共会有2460场比赛，对应地，就会有4920个`Team`实例。但是，NBA的30支球队是固定的，实际上只需30个`Team`实例就能完整地记录一个赛季的所有比赛信息，剩下的4890个`Team`实例属于冗余的数据。

这种场景下就适合采用享元模式来进行优化，我们把`Team`设计成多个`Match`实例之间的享元。享元的获取通过享元工厂来完成，享元工厂`teamFactory`的定义如下，Client统一使用`teamFactory.TeamOf`方法来获取球队`Team`实例。其中，每个球队`Team`实例只会创建一次，然后添加到球队池中，后续获取都是直接从池中获取，这样就达到了共享的目的。

```go
package nba
...
type teamFactory struct {
	// 球队池，缓存球队实例
	teams map[TeamId]*Team
}

// 根据TeamId获取Team实例，从池中获取，如果池里没有，则创建
func (t *teamFactory) TeamOf(id TeamId) *Team {
	team, ok := t.teams[id]
	if !ok {
		team = createTeam(id)
		t.teams[id] = team
	}
	return team
}

// 享元工厂的单例
var factory = &teamFactory{
	teams: make(map[TeamId]*Team),
}

func Factory() *teamFactory {
	return factory
}

// 根据TeamId创建Team实例，只在TeamOf方法中调用，外部不可见
func createTeam(id TeamId) *Team {
	switch id {
	case Warrior:
		w := &Team{
			Id:      Warrior,
			Name:    "Golden State Warriors",
		}
		curry := &Player{
			Name: "Stephen Curry",
			Team: Warrior,
		}
		thompson := &Player{
			Name: "Klay Thompson",
			Team: Warrior,
		}
		w.Players = append(w.Players, curry, thompson)
		return w
	case Laker:
		l := &Team{
			Id:      Laker,
			Name:    "Los Angeles Lakers",
		}
		james := &Player{
			Name: "LeBron James",
			Team: Laker,
		}
		davis := &Player{
			Name: "Anthony Davis",
			Team: Laker,
		}
		l.Players = append(l.Players, james, davis)
		return l
	default:
		fmt.Printf("Get an invalid team id %v.\n", id)
		return nil
	}
}
```

测试代码如下：

```go
package test
...
func TestFlyweight(t *testing.T) {
	game1 := &nba.Match{
		Date:         time.Date(2020, 1, 10, 9, 30, 0, 0, time.Local),
		LocalTeam:    nba.Factory().TeamOf(nba.Warrior),
		VisitorTeam:  nba.Factory().TeamOf(nba.Laker),
		LocalScore:   102,
		VisitorScore: 99,
	}
	game1.ShowResult()
	game2 := &nba.Match{
		Date:         time.Date(2020, 1, 12, 9, 30, 0, 0, time.Local),
		LocalTeam:    nba.Factory().TeamOf(nba.Laker),
		VisitorTeam:  nba.Factory().TeamOf(nba.Warrior),
		LocalScore:   110,
		VisitorScore: 118,
	}
	game2.ShowResult()
  // 两个Match的同一个球队应该是同一个实例的
	if game1.LocalTeam != game2.VisitorTeam {
		t.Errorf("Warrior team do not use flyweight pattern")
	}
}
// 运行结果
=== RUN   TestFlyweight
Golden State Warriors VS Los Angeles Lakers - 102:99
Los Angeles Lakers VS Golden State Warriors - 110:118
--- PASS: TestFlyweight (0.00s)
```

## 总结

本文我们主要介绍了结构型模式中的代理模式、装饰模式、外观模式和享元模式。**代理模式**为一个对象提供一种代理以控制对该对象的访问，强调的是对本体对象的访问控制；**装饰模式**能够动态地为本体对象叠加新的行为，强调的是为本体对象添加新的功能；**外观模式**为子系统提供了一个更高层次的对外统一接口，强调的是分层和解耦；**享元模式**通过共享对象来降低系统的资源消耗，强调的是如何在多个对象中共享相同的状态。

到目前为止，7种结构型模式已经全部介绍完，下一篇文章，我们开始将介绍最后一类设计模式——**行为型模式**（Behavioral Pattern）。
