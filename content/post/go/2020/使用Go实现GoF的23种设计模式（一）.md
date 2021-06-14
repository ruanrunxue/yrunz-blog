---
title: 使用Go实现GoF的23种设计模式（一）
date: 2020-08-10
categories:
 - Golang
tags:
 - golang
 - 设计模式
---

## 前言

从1995年GoF提出23种**设计模式**到现在，25年过去了，设计模式依旧是软件领域的热门话题。在当下，如果你不会一点设计模式，都不好意思说自己是一个合格的程序员。设计模式通常被定义为：

> 设计模式（Design Pattern）是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结，使用设计模式是为了可重用代码、让代码更容易被他人理解并且保证代码可靠性。

从定义上看，**设计模式其实是一种经验的总结，是针对特定问题的简洁而优雅的解决方案**。既然是经验总结，那么学习设计模式最直接的好处就在于可以站在巨人的肩膀上解决软件开发过程中的一些特定问题。然而，学习设计模式的最高境界是习得其中解决问题所用到的思想，当你把它们的本质思想吃透了，也就能做到**即使已经忘掉某个设计模式的名称和结构，也能在解决特定问题时信手拈来**。

好的东西有人吹捧，当然也会招黑。设计模式被抨击主要因为以下两点：

1、*设计模式会增加代码量，把程序逻辑变得复杂*。这一点是不可避免的，但是我们并不能仅仅只考虑开发阶段的成本。最简单的程序当然是一个函数从头写到尾，但是这样后期的维护成本会变得非常大；而设计模式虽然增加了一点开发成本，但是能让人们写出可复用、可维护性高的程序。引用《软件设计的哲学》里的概念，前者就是**战术编程**，后者就是**战略编程**，我们应该**对战术编程Say No**！（请移步[《一步步降低软件复杂性》](https://www.yrunz.com/archives/一步步降低软件复杂性)）

2、*滥用设计模式*。这是初学者最容易犯的错误，当学到一个模式时，恨不得在所有的代码都用上，从而在不该使用模式的地方刻意地使用了模式，导致了程序变得异常复杂。其实每个设计模式都有几个关键要素：**适用场景**、**解决方法**、**优缺点**。模式并不是万能药，它只有在特定的问题上才能显现出效果。所以，在使用一个模式前，先问问自己，当前的这个场景适用这个模式吗？

《设计模式》一书的副标题是“可复用面向对象软件的基础”，但并不意味着只有面向对象语言才能使用设计模式。模式只是一种解决特定问题的思想，跟语言无关。就像Go语言一样，它并非是像C++和Java一样的面向对象语言，但是设计模式同样适用。本系列文章将使用Go语言来实现GoF提出的23种设计模式，按照**创建型模式**（Creational Pattern）、**结构型模式**（Structural Pattern）和**行为型模式**（Behavioral Pattern）三种类别进行组织，文本主要介绍其中的创建型模式。

## 单例模式（Singleton Pattern）

![单例模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghky3yanabj318q0iwnpd.jpg)

### 简述

单例模式算是23中设计模式里最简单的一个了，它主要用于**保证一个类仅有一个实例，并提供一个访问它的全局访问点**。

在程序设计中，有一些对象通常我们只需要一个共享的实例，比如线程池、全局缓存、对象池等，这种场景下就适合使用单例模式。

但是，并非所有全局唯一的场景都适合使用单例模式。比如，考虑需要统计一个API调用的情况，有两个指标，成功调用次数和失败调用次数。这两个指标都是全局唯一的，所以有人可能会将其建模成两个单例`SuccessApiMetric`和`FailApiMetric`。按照这个思路，随着指标数量的增多，你会发现代码里类的定义会越来越多，也越来越臃肿。这也是单例模式最常见的误用场景，更好的方法是将两个指标设计成一个对象`ApiMetric`下的两个实例`ApiMetic success`和`ApiMetic fail`。

*如何判断一个对象是否应该被建模成单例？*

通常，被建模成单例的对象都有“**中心点**”的含义，比如线程池就是管理所有线程的中心。所以，在判断一个对象是否适合单例模式时，先思考下，这个对象是一个中心点吗？

### Go实现

在对某个对象实现单例模式时，有两个点必须要注意：（1）**限制调用者直接实例化该对象**；（2）**为该对象的单例提供一个全局唯一的访问方法**。

对于C++/Java而言，只需把类的构造函数设计成私有的，并提供一个`static`方法去访问该类点唯一实例即可。但对于Go语言来说，即没有构造函数的概念，也没有`static`方法，所以需要另寻出路。

我们可以利用Go语言`package`的访问规则来实现，将单例结构体设计成首字母小写，就能限定其访问范围只在当前package下，模拟了C++/Java中的私有构造函数；再在当前`package`下实现一个首字母大写的访问函数，就相当于`static`方法的作用了。

在实际开发中，我们经常会遇到需要频繁创建和销毁的对象。频繁的创建和销毁一则消耗CPU，二则内存的利用率也不高，通常我们都会使用对象池技术来进行优化。考虑我们需要实现一个消息对象池，因为是全局的中心点，管理所有的Message实例，所以将其实现成单例，实现代码如下：

```go
package msgpool
...
// 消息池
type messagePool struct {
	pool *sync.Pool
}
// 消息池单例
var msgPool = &messagePool{
	// 如果消息池里没有消息，则新建一个Count值为0的Message实例
	pool: &sync.Pool{New: func() interface{} { return &Message{Count: 0} }},
}
// 访问消息池单例的唯一方法
func Instance() *messagePool {
	return msgPool
}
// 往消息池里添加消息
func (m *messagePool) AddMsg(msg *Message) {
	m.pool.Put(msg)
}
// 从消息池里获取消息
func (m *messagePool) GetMsg() *Message {
	return m.pool.Get().(*Message)
}
...
```

测试代码如下：

```go
package test
...
func TestMessagePool(t *testing.T) {
	msg0 := msgpool.Instance().GetMsg()
	if msg0.Count != 0 {
		t.Errorf("expect msg count %d, but actual %d.", 0, msg0.Count)
	}
	msg0.Count = 1
	msgpool.Instance().AddMsg(msg0)
	msg1 := msgpool.Instance().GetMsg()
	if msg1.Count != 1 {
		t.Errorf("expect msg count %d, but actual %d.", 1, msg1.Count)
	}
}
// 运行结果
=== RUN   TestMessagePool
--- PASS: TestMessagePool (0.00s)
PASS
```

以上的单例模式就是典型的“**饿汉模式**”，实例在系统加载的时候就已经完成了初始化。对应地，还有一种“**懒汉模式**”，只有等到对象被使用的时候，才会去初始化它，从而一定程度上节省了内存。众所周知，“懒汉模式”会带来线程安全问题，可以通过**普通加锁**，或者更高效的**双重检验锁**来优化。对于“懒汉模式”，Go语言有一个更优雅的实现方式，那就是利用`sync.Once`，它有一个`Do`方法，其入参是一个方法，Go语言会保证仅仅只调用一次该方法。

```go
// 单例模式的“懒汉模式”实现
package msgpool
...
var once = &sync.Once{}
// 消息池单例，在首次调用时初始化
var msgPool *messagePool
// 全局唯一获取消息池pool到方法
func Instance() *messagePool {
	// 在匿名函数中实现初始化逻辑，Go语言保证只会调用一次
	once.Do(func() {
		msgPool = &messagePool{
			// 如果消息池里没有消息，则新建一个Count值为0的Message实例
			pool: &sync.Pool{New: func() interface{} { return &Message{Count: 0} }},
		}
	})
	return msgPool
}
...
```

## 建造者模式（Builder Pattern）

![建造者模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghky4kprezj319e0kuu0x.jpg)

### 简述

在程序设计中，我们会经常遇到一些复杂的对象，其中有很多成员属性，甚至嵌套着多个复杂的对象。这种情况下，创建这个复杂对象就会变得很繁琐。对于C++/Java而言，最常见的表现就是构造函数有着长长的参数列表：

```java
MyObject obj = new MyObject(param1, param2, param3, param4, param5, param6, ...)
```

而对于Go语言来说，最常见的表现就是多层的嵌套实例化：

```go
obj := &MyObject{
  Field1: &Field1 {
    Param1: &Param1 {
      Val: 0,
    },
    Param2: &Param2 {
      Val: 1,
    },
    ...
  },
  Field2: &Field2 {
    Param3: &Param3 {
      Val: 2,
    },
    ...
  },
  ...
}
```

上述的对象创建方法有两个明显的缺点：（1）**对对象使用者不友好**，使用者在创建对象时需要知道的细节太多；（2）**代码可读性很差**。

*针对这种对象成员较多，创建对象逻辑较为繁琐的场景，就适合使用建造者模式来进行优化。*

建造者模式的作用有如下几个：

1、封装复杂对象的创建过程，使对象使用者不感知复杂的创建逻辑。

2、可以一步步按照顺序对成员进行赋值，或者创建嵌套对象，并最终完成目标对象的创建。

3、对多个对象复用同样的对象创建逻辑。

其中，第1和第2点比较常用，下面对建造者模式的实现也主要是针对这两点进行示例。

### Go实现

考虑如下的一个`Message`结构体，其主要有`Header`和`Body`组成：

```go
package msg
...
type Message struct {
	Header *Header
	Body   *Body
}
type Header struct {
	SrcAddr  string
	SrcPort  uint64
	DestAddr string
	DestPort uint64
	Items    map[string]string
}
type Body struct {
	Items []string
}
...
```

如果按照直接的对象创建方式，创建逻辑应该是这样的：

```go
// 多层的嵌套实例化
message := msg.Message{
	Header: &msg.Header{
		SrcAddr:  "192.168.0.1",
		SrcPort:  1234,
		DestAddr: "192.168.0.2",
		DestPort: 8080,
		Items:    make(map[string]string),
	},
	Body:   &msg.Body{
		Items: make([]string, 0),
	},
}
// 需要知道对象的实现细节
message.Header.Items["contents"] = "application/json"
message.Body.Items = append(message.Body.Items, "record1")
message.Body.Items = append(message.Body.Items, "record2")
```

虽然`Message`结构体嵌套的层次不多，但是从其创建的代码来看，确实存在**对对象使用者不友好**和**代码可读性差**的缺点。下面我们引入建造者模式对代码进行重构：

```go
package msg
...
// Message对象的Builder对象
type builder struct {
	once *sync.Once
	msg *Message
}
// 返回Builder对象
func Builder() *builder {
	return &builder{
		once: &sync.Once{},
		msg: &Message{Header: &Header{}, Body: &Body{}},
	}
}
// 以下是对Message成员对构建方法
func (b *builder) WithSrcAddr(srcAddr string) *builder {
	b.msg.Header.SrcAddr = srcAddr
	return b
}
func (b *builder) WithSrcPort(srcPort uint64) *builder {
	b.msg.Header.SrcPort = srcPort
	return b
}
func (b *builder) WithDestAddr(destAddr string) *builder {
	b.msg.Header.DestAddr = destAddr
	return b
}
func (b *builder) WithDestPort(destPort uint64) *builder {
	b.msg.Header.DestPort = destPort
	return b
}
func (b *builder) WithHeaderItem(key, value string) *builder {
  // 保证map只初始化一次
	b.once.Do(func() {
		b.msg.Header.Items = make(map[string]string)
	})
	b.msg.Header.Items[key] = value
	return b
}
func (b *builder) WithBodyItem(record string) *builder {
	b.msg.Body.Items = append(b.msg.Body.Items, record)
	return b
}
// 创建Message对象，在最后一步调用
func (b *builder) Build() *Message {
	return b.msg
}
```

测试代码如下：

```go
package test
...
func TestMessageBuilder(t *testing.T) {
  // 使用消息建造者进行对象创建
	message := msg.Builder().
		WithSrcAddr("192.168.0.1").
		WithSrcPort(1234).
		WithDestAddr("192.168.0.2").
		WithDestPort(8080).
		WithHeaderItem("contents", "application/json").
		WithBodyItem("record1").
		WithBodyItem("record2").
		Build()
	if message.Header.SrcAddr != "192.168.0.1" {
		t.Errorf("expect src address 192.168.0.1, but actual %s.", message.Header.SrcAddr)
	}
	if message.Body.Items[0] != "record1" {
		t.Errorf("expect body item0 record1, but actual %s.", message.Body.Items[0])
	}
}
// 运行结果
=== RUN   TestMessageBuilder
--- PASS: TestMessageBuilder (0.00s)
PASS
```

从测试代码可知，使用建造者模式来进行对象创建，使用者不再需要知道对象具体的实现细节，代码可读性也更好。

## 工厂方法模式（Factory Method Pattern）

![工厂方法模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghkpq8hayrj31cm0sskjm.jpg)

### 简述

 工厂方法模式跟上一节讨论的建造者模式类似，都是**将对象创建的逻辑封装起来，为使用者提供一个简单易用的对象创建接口**。两者在应用场景上稍有区别，建造者模式更常用于需要传递多个参数来进行实例化的场景。

使用工厂方法来创建对象主要有两个好处：

1、**代码可读性更好**。相比于使用C++/Java中的构造函数，或者Go中的`{}`来创建对象，工厂方法因为可以通过函数名来表达代码含义，从而具备更好的可读性。比如，使用工厂方法`productA := CreateProductA()`创建一个`ProductA`对象，比直接使用`productA := ProductA{}`的可读性要好。

2、**与使用者代码解耦**。很多情况下，对象的创建往往是一个容易变化的点，通过工厂方法来封装对象的创建过程，可以在创建逻辑变更时，避免**霰弹式修改**。

工厂方法模式也有两种实现方式：（1）提供一个工厂对象，通过调用工厂对象的工厂方法来创建产品对象；（2）将工厂方法集成到产品对象中（C++/Java中对象的`static`方法，Go中同一`package`下的函数）

### Go实现

考虑有一个事件对象`Event`，分别有两种有效的时间类型`Start`和`End`：

```go
package event
...
type Type uint8
// 事件类型定义
const (
	Start Type = iota
	End
)
// 事件抽象接口
type Event interface {
	EventType() Type
	Content() string
}
// 开始事件，实现了Event接口
type StartEvent struct{
	content string
}
...
// 结束事件，实现了Event接口
type EndEvent struct{
	content string
}
...
```

1、按照第一种实现方式，为`Event`提供一个工厂对象，具体代码如下：

```go
package event
...
// 事件工厂对象
type Factory struct{}
// 更具事件类型创建具体事件
func (e *Factory) Create(etype Type) Event {
	switch etype {
	case Start:
		return &StartEvent{
			content: "this is start event",
		}
	case End:
		return &EndEvent{
			content: "this is end event",
		}
	default:
		return nil
	}
}
```

测试代码如下：

```go
package test
...
func TestEventFactory(t *testing.T) {
	factory := event.Factory{}
	e := factory.Create(event.Start)
	if e.EventType() != event.Start {
		t.Errorf("expect event.Start, but actual %v.", e.EventType())
	}
	e = factory.Create(event.End)
	if e.EventType() != event.End {
		t.Errorf("expect event.End, but actual %v.", e.EventType())
	}
}
// 运行结果
=== RUN   TestEventFactory
--- PASS: TestEventFactory (0.00s)
PASS
```

2、按照第二种实现方式，分别给`Start`和`End`类型的`Event`单独提供一个工厂方法，代码如下：

```go
package event
...
// Start类型Event的工厂方法
func OfStart() Event {
	return &StartEvent{
		content: "this is start event",
	}
}
// End类型Event的工厂方法
func OfEnd() Event {
	return &EndEvent{
		content: "this is end event",
	}
}
```

测试代码如下：

```go
package event
...
func TestEvent(t *testing.T) {
	e := event.OfStart()
	if e.EventType() != event.Start {
		t.Errorf("expect event.Start, but actual %v.", e.EventType())
	}
	e = event.OfEnd()
	if e.EventType() != event.End {
		t.Errorf("expect event.End, but actual %v.", e.EventType())
	}
}
// 运行结果
=== RUN   TestEvent
--- PASS: TestEvent (0.00s)
PASS
```

## 抽象工厂模式（Abstract Factory Pattern）

![抽象工厂模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghkxo5tf5ij31ak0om4qq.jpg)

### 简述

在工厂方法模式中，我们通过一个工厂对象来创建一个产品族，具体创建哪个产品，则通过`swtich-case`的方式去判断。这也意味着该产品组上，每新增一类产品对象，都必须修改原来工厂对象的代码；而且随着产品的不断增多，工厂对象的职责也越来越重，违反了**单一职责原则**。

抽象工厂模式通过给工厂类新增一个抽象层解决了该问题，如上图所示，`FactoryA`和`FactoryB`都实现·抽象工厂接口，分别用于创建`ProductA`和`ProductB`。如果后续新增了`ProductC`，只需新增一个`FactoryC`即可，无需修改原有的代码；因为每个工厂只负责创建一个产品，因此也遵循了**单一职责原则**。

### Go实现

考虑需要如下一个插件架构风格的消息处理系统，`pipeline`是消息处理的管道，其中包含了`input`、`filter`和`output`三个插件。我们需要实现根据配置来创建`pipeline` ，加载插件过程的实现非常适合使用工厂模式，其中`input`、`filter`和`output`三类插件的创建使用抽象工厂模式，而`pipeline`的创建则使用工厂方法模式。

![插件化风格的消息处理系统](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghkw23e4r3j31bs0nge82.jpg)

各类插件和`pipeline`的接口定义如下：

```go
package plugin
...
// 插件抽象接口定义
type Plugin interface {}
// 输入插件，用于接收消息
type Input interface {
	Plugin
	Receive() string
}
// 过滤插件，用于处理消息
type Filter interface {
	Plugin
	Process(msg string) string
}
// 输出插件，用于发送消息
type Output interface {
	Plugin
	Send(msg string)
}
```

```go
package pipeline
...
// 消息管道的定义
type Pipeline struct {
	input  plugin.Input
	filter plugin.Filter
	output plugin.Output
}
// 一个消息的处理流程为 input -> filter -> output
func (p *Pipeline) Exec() {
	msg := p.input.Receive()
	msg = p.filter.Process(msg)
	p.output.Send(msg)
}
```

接着，我们定义`input`、`filter`、`output`三类插件接口的具体实现：

```go
package plugin
...
// input插件名称与类型的映射关系，主要用于通过反射创建input对象
var inputNames = make(map[string]reflect.Type)
// Hello input插件，接收“Hello World”消息
type HelloInput struct {}

func (h *HelloInput) Receive() string {
	return "Hello World"
}
// 初始化input插件映射关系表
func init() {
	inputNames["hello"] = reflect.TypeOf(HelloInput{})
}
```

```go
package plugin
...
// filter插件名称与类型的映射关系，主要用于通过反射创建filter对象
var filterNames = make(map[string]reflect.Type)
// Upper filter插件，将消息全部字母转成大写
type UpperFilter struct {}

func (u *UpperFilter) Process(msg string) string {
	return strings.ToUpper(msg)
}
// 初始化filter插件映射关系表
func init() {
	filterNames["upper"] = reflect.TypeOf(UpperFilter{})
}
```

```go
package plugin
...
// output插件名称与类型的映射关系，主要用于通过反射创建output对象
var outputNames = make(map[string]reflect.Type)
// Console output插件，将消息输出到控制台上
type ConsoleOutput struct {}

func (c *ConsoleOutput) Send(msg string) {
	fmt.Println(msg)
}
// 初始化output插件映射关系表
func init() {
	outputNames["console"] = reflect.TypeOf(ConsoleOutput{})
}
```

然后，我们定义插件抽象工厂接口，以及对应插件的工厂实现：

```go
package plugin
...
// 插件抽象工厂接口
type Factory interface {
	Create(conf Config) Plugin
}
// input插件工厂对象，实现Factory接口
type InputFactory struct{}
// 读取配置，通过反射机制进行对象实例化
func (i *InputFactory) Create(conf Config) Plugin {
	t, _ := inputNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}
// filter和output插件工厂实现类似
type FilterFactory struct{}
func (f *FilterFactory) Create(conf Config) Plugin {
	t, _ := filterNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}
type OutputFactory struct{}
func (o *OutputFactory) Create(conf Config) Plugin {
	t, _ := outputNames[conf.Name]
	return reflect.New(t).Interface().(Plugin)
}
```

最后定义`pipeline`的工厂方法，调用`plugin.Factory`抽象工厂完成pipelien对象的实例化：

```go
package pipeline
...
// 保存用于创建Plugin的工厂实例，其中map的key为插件类型，value为抽象工厂接口
var pluginFactories = make(map[plugin.Type]plugin.Factory)
// 根据plugin.Type返回对应Plugin类型的工厂实例
func factoryOf(t plugin.Type) plugin.Factory {
	factory, _ := pluginFactories[t]
	return factory
}
// pipeline工厂方法，根据配置创建一个Pipeline实例
func Of(conf Config) *Pipeline {
	p := &Pipeline{}
	p.input = factoryOf(plugin.InputType).Create(conf.Input).(plugin.Input)
	p.filter = factoryOf(plugin.FilterType).Create(conf.Filter).(plugin.Filter)
	p.output = factoryOf(plugin.OutputType).Create(conf.Output).(plugin.Output)
	return p
}
// 初始化插件工厂对象
func init() {
	pluginFactories[plugin.InputType] = &plugin.InputFactory{}
	pluginFactories[plugin.FilterType] = &plugin.FilterFactory{}
	pluginFactories[plugin.OutputType] = &plugin.OutputFactory{}
}
```

测试代码如下：

```go
package test
...
func TestPipeline(t *testing.T) {
  // 其中pipeline.DefaultConfig()的配置内容见【抽象工厂模式示例图】
  // 消息处理流程为 HelloInput -> UpperFilter -> ConsoleOutput
	p := pipeline.Of(pipeline.DefaultConfig())
	p.Exec()
}
// 运行结果
=== RUN   TestPipeline
HELLO WORLD
--- PASS: TestPipeline (0.00s)
PASS
```

## 原型模式（Prototype Pattern）

![原型模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghky39ichjj319u0gqhdt.jpg)

### 简述

原型模式主要解决对象复制的问题，它的核心就是`clone()`方法，返回`Prototype`对象的复制品。在程序设计过程中，往往会遇到有一些场景需要大量相同的对象，如果不使用原型模式，那么我们可能会这样进行对象的创建：*新创建一个相同对象的实例，然后遍历原始对象的所有成员变量， 并将成员变量值复制到新对象中*。这种方法的缺点很明显，那就是使用者必须知道对象的实现细节，导致代码之间的耦合。另外，对象很有可能存在除了对象本身以外不可见的变量，这种情况下该方法就行不通了。

对于这种情况，更好的方法就是使用原型模式，将复制逻辑委托给对象本身，这样，上述两个问题也都迎刃而解了。

### Go实现

还是以建造者模式一节中的`Message`作为例子，现在设计一个`Prototype`抽象接口：

```go
package prototype
...
// 原型复制抽象接口
type Prototype interface {
	clone() Prototype
}

type Message struct {
	Header *Header
	Body   *Body
}

func (m *Message) clone() Prototype {
	msg := *m
	return &msg
}
```

测试代码如下：

```go
package test
...
func TestPrototype(t *testing.T) {
	message := msg.Builder().
		WithSrcAddr("192.168.0.1").
		WithSrcPort(1234).
		WithDestAddr("192.168.0.2").
		WithDestPort(8080).
		WithHeaderItem("contents", "application/json").
		WithBodyItem("record1").
		WithBodyItem("record2").
		Build()
  // 复制一份消息
	newMessage := message.Clone().(*msg.Message)
	if newMessage.Header.SrcAddr != message.Header.SrcAddr {
		t.Errorf("Clone Message failed.")
	}
	if newMessage.Body.Items[0] != message.Body.Items[0] {
		t.Errorf("Clone Message failed.")
	}
}
// 运行结果
=== RUN   TestPrototype
--- PASS: TestPrototype (0.00s)
PASS
```

## 总结

本文主要介绍了GoF的23种设计模式中的5种创建型模式，创建型模式的目的都是**提供一个简单的接口，让对象的创建过程与使用者解耦**。其中，**单例模式**主要用于保证一个类仅有一个实例，并提供一个访问它的全局访问点；**建造者模式**主要解决需要创建对象时需要传入多个参数，或者对初始化顺序有要求的场景；**工厂方法模式**通过提供一个工厂对象或者工厂方法，为使用者隐藏了对象创建的细节；**抽象工厂模式**是对工厂方法模式的优化，通过为工厂对象新增一个抽象层，让工厂对象遵循单一职责原则，也避免了霰弹式修改；**原型模式**则让对象复制更加简单。

下一篇文章，将介绍23种设计模式中的7种**结构型模式**（Structural Pattern），及其Go语言的实现。

