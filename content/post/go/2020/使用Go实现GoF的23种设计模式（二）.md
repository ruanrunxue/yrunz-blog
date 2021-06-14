---
title: 使用Go实现GoF的23种设计模式（二）
date: 2020-08-23
categories:
 - Golang
tags:
 - golang
 - 设计模式
---

## 前言

上一篇文章[《使用Go实现GoF的23种设计模式（一）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（一）)介绍了23种设计模式中的**创建型模式**（Creational Pattern），创建型模式是处理对象创建的一类设计模式，主要思想是**向对象的使用者隐藏对象创建的具体细节**，从而达到解耦的目的。本文主要聚焦在**结构型模式**（Structural Pattern）上，其主要思想是**将多个对象组装成较大的结构，并同时保持结构的灵活和高效**，从程序的结构上解决模块之间的耦合问题。

## 组合模式（Composite Pattern）

![组合模式结构](https://tva1.sinaimg.cn/large/007S8ZIlly1ghvdqutjd5j312k0iue81.jpg)

### 简述

在面向对象编程中，有两个常见的对象设计方法，**组合**和**继承**，两者都可以解决代码复用的问题，但是使用后者时容易出现继承层次过深，对象关系过于复杂的副作用，从而导致代码的可维护性变差。因此，一个经典的面向对象设计原则是：**组合优于继承**。

我们都知道，组合所表示的语义为“has-a”，也就是部分和整体的关系，最经典的组合模式描述如下：

> 将对象组合成树形结构以表示“部分-整体”的层次结构，使得用户对单个对象和组合对象的使用具有一致性。

Go语言天然就支持了组合模式，而且从它不支持继承关系的特点来看，Go也奉行了**组合优于继承**的原则，鼓励大家在进行程序设计时多采用组合的方法。Go实现组合模式的方式有两种，分别是**直接组合**（Direct Composition）和**嵌入组合**（Embedding Composition），下面我们一起探讨这两种不同的实现方法。

### Go实现

*直接组合（Direct Composition）的实现方式类似于Java/C++，就是将一个对象作为另一个对象的成员属性。*

一个典型的实现如[《使用Go实现GoF的23种设计模式（一）》](https://www.yrunz.com/archives/使用Go实现GoF的23种设计模式（一）)中所举的例子，一个`Message`结构体，由`Header`和`Body`所组成。那么`Message`就是一个整体，而`Header`和`Body`则为消息的组成部分。

```go
type Message struct {
	Header *Header
	Body   *Body
}
```

现在，我们来看一个稍微复杂一点的例子，同样考虑上一篇文章中所描述的插件架构风格的消息处理系统。前面我们用**抽象工厂模式**解决了插件加载的问题，通常，每个插件都会有一个生命周期，常见的就是启动状态和停止状态，现在我们使用**组合模式**来解决插件的启动和停止问题。

首先给`Plugin`接口添加几个生命周期相关的方法：

```go
package plugin
...
// 插件运行状态
type Status uint8

const (
	Stopped Status = iota
	Started
)

type Plugin interface {
  // 启动插件
	Start()
  // 停止插件
	Stop()
  // 返回插件当前的运行状态
	Status() Status
}
// Input、Filter、Output三类插件接口的定义跟上一篇文章类似
// 这里使用Message结构体替代了原来的string，使得语义更清晰
type Input interface {
	Plugin
	Receive() *msg.Message
}

type Filter interface {
	Plugin
	Process(msg *msg.Message) *msg.Message
}

type Output interface {
	Plugin
	Send(msg *msg.Message)
}

```

对于插件化的消息处理系统而言，一切皆是插件，因此我们将`Pipeine`也设计成一个插件，实现`Plugin`接口：

```go
package pipeline
...
// 一个Pipeline由input、filter、output三个Plugin组成
type Pipeline struct {
	status plugin.Status
	input  plugin.Input
	filter plugin.Filter
	output plugin.Output
}

func (p *Pipeline) Exec() {
	msg := p.input.Receive()
	msg = p.filter.Process(msg)
	p.output.Send(msg)
}
// 启动的顺序 output -> filter -> input
func (p *Pipeline) Start() {
	p.output.Start()
	p.filter.Start()
	p.input.Start()
	p.status = plugin.Started
	fmt.Println("Hello input plugin started.")
}
// 停止的顺序 input -> filter -> output
func (p *Pipeline) Stop() {
	p.input.Stop()
	p.filter.Stop()
	p.output.Stop()
	p.status = plugin.Stopped
	fmt.Println("Hello input plugin stopped.")
}

func (p *Pipeline) Status() plugin.Status {
	return p.status
}

```

一个`Pipeline`由`Input`、`Filter`、`Output`三类插件组成，形成了“部分-整体”的关系，而且它们都实现了`Plugin`接口，这就是一个典型的组合模式的实现。Client无需显式地启动和停止`Input`、`Filter`和`Output`插件，在调用`Pipeline`对象的`Start`和`Stop`方法时，`Pipeline`就已经帮你按顺序完成对应插件的启动和停止。

相比于上一篇文章，在本文中实现`Input`、`Filter`、`Output`三类插件时，需要多实现3个生命周期的方法。还是以上一篇文章中的`HelloInput`、`UpperFilter`和`ConsoleOutput`作为例子，具体实现如下：

```go
package plugin
...
type HelloInput struct {
	status Status
}

func (h *HelloInput) Receive() *msg.Message {
  // 如果插件未启动，则返回nil
	if h.status != Started {
		fmt.Println("Hello input plugin is not running, input nothing.")
		return nil
	}
	return msg.Builder().
		WithHeaderItem("content", "text").
		WithBodyItem("Hello World").
		Build()
}

func (h *HelloInput) Start() {
	h.status = Started
	fmt.Println("Hello input plugin started.")
}

func (h *HelloInput) Stop() {
	h.status = Stopped
	fmt.Println("Hello input plugin stopped.")
}

func (h *HelloInput) Status() Status {
	return h.status
}
```

```go
package plugin
...
type UpperFilter struct {
	status Status
}

func (u *UpperFilter) Process(msg *msg.Message) *msg.Message {
	if u.status != Started {
		fmt.Println("Upper filter plugin is not running, filter nothing.")
		return msg
	}
	for i, val := range msg.Body.Items {
		msg.Body.Items[i] = strings.ToUpper(val)
	}
	return msg
}

func (u *UpperFilter) Start() {
	u.status = Started
	fmt.Println("Upper filter plugin started.")
}

func (u *UpperFilter) Stop() {
	u.status = Stopped
	fmt.Println("Upper filter plugin stopped.")
}

func (u *UpperFilter) Status() Status {
	return u.status
}

```

```go
package plugin
...
type ConsoleOutput struct {
	status Status
}

func (c *ConsoleOutput) Send(msg *msg.Message) {
	if c.status != Started {
		fmt.Println("Console output is not running, output nothing.")
		return
	}
	fmt.Printf("Output:\n\tHeader:%+v, Body:%+v\n", msg.Header.Items, msg.Body.Items)
}

func (c *ConsoleOutput) Start() {
	c.status = Started
	fmt.Println("Console output plugin started.")
}

func (c *ConsoleOutput) Stop() {
	c.status = Stopped
	fmt.Println("Console output plugin stopped.")
}

func (c *ConsoleOutput) Status() Status {
	return c.status
}

```

测试代码如下：

```go
package test
...
func TestPipeline(t *testing.T) {
	p := pipeline.Of(pipeline.DefaultConfig())
	p.Start()
	p.Exec()
	p.Stop()
}
// 运行结果
=== RUN   TestPipeline
Console output plugin started.
Upper filter plugin started.
Hello input plugin started.
Pipeline started.
Output:
    Header:map[content:text], Body:[HELLO WORLD]
Hello input plugin stopped.
Upper filter plugin stopped.
Console output plugin stopped.
Hello input plugin stopped.
--- PASS: TestPipeline (0.00s)
PASS
```

*组合模式的另一种实现，嵌入组合（Embedding Composition），其实就是利用了Go语言的匿名成员特性，本质上跟直接组合是一致的。*

还是以Message结构体为例，如果采用嵌入组合，则看起来像是这样：

```go
type Message struct {
	Header
	Body
}
// 使用时，Message可以引用Header和Body的成员属性，例如：
msg := &Message{}
msg.SrcAddr = "192.168.0.1"
```

## 适配器模式（Adapter Pattern）

![适配器模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghzyia5t19j315w0k0kjl.jpg)

### 简述

适配器模式是最常用的结构型模式之一，它让原本因为接口不匹配而无法一起工作的两个对象能够一起工作。在现实生活中，适配器模式也是处处可见，比如电源插头转换器，可以让英式的插头工作在中式的插座上。适配器模式所做的就是**将一个接口`Adaptee`，通过适配器`Adapter`转换成Client所期望的另一个接口`Target`来使用**，实现原理也很简单，就是`Adapter`通过实现`Target`接口，并在对应的方法中调用`Adaptee`的接口实现。

一个典型的应用场景是，*系统中一个老的接口已经过时即将废弃，但因为历史包袱没法立即将老接口全部替换为新接口，这时可以新增一个适配器，将老的接口适配成新的接口来使用*。适配器模式很好的践行了面向对象设计原则里的**开闭原则**（open/closed principle），新增一个接口时也无需修改老接口，只需多加一个适配层即可。

### Go实现

继续考虑上一节的消息处理系统例子，目前为止，系统的输入都源自于`HelloInput`，现在假设需要给系统新增从Kafka消息队列中接收数据的功能，其中Kafka消费者的接口如下：

```go
package kafka
...
type Records struct {
	Items []string
}

type Consumer interface {
	Poll() Records
}
```

由于当前`Pipeline`的设计是通过`plugin.Input`接口来进行数据接收，因此`kafka.Consumer`并不能直接集成到系统中。

*怎么办？使用适配器模式！*

为了能让`Pipeline`能够使用`kafka.Consumer`接口，我们需要定义一个适配器如下：

```go
package plugin
...
type KafkaInput struct {
	status Status
	consumer kafka.Consumer
}

func (k *KafkaInput) Receive() *msg.Message {
	records := k.consumer.Poll()
	if k.status != Started {
		fmt.Println("Kafka input plugin is not running, input nothing.")
		return nil
	}
	return msg.Builder().
		WithHeaderItem("content", "text").
		WithBodyItems(records.Items).
		Build()
}

// 在输入插件映射关系中加入kafka，用于通过反射创建input对象
func init() {
	inputNames["hello"] = reflect.TypeOf(HelloInput{})
	inputNames["kafka"] = reflect.TypeOf(KafkaInput{})
}
...
```

因为Go语言并没有构造函数，如果按照上一篇文章中的**抽象工厂模式**来创建`KafkaInput`，那么得到的实例中的`consumer`成员因为没有被初始化而会是`nil`。因此，需要给`Plugin`接口新增一个`Init`方法，用于定义插件的一些初始化操作，并在工厂返回实例前调用。

```go
package plugin
...
type Plugin interface {
	Start()
	Stop()
	Status() Status
	// 新增初始化方法，在插件工厂返回实例前调用
	Init()
}

// 修改后的插件工厂实现如下
func (i *InputFactory) Create(conf Config) Plugin {
	t, _ := inputNames[conf.Name]
	p := reflect.New(t).Interface().(Plugin)
  // 返回插件实例前调用Init函数，完成相关初始化方法
	p.Init()
	return p
}

// KakkaInput的Init函数实现
func (k *KafkaInput) Init() {
	k.consumer = &kafka.MockConsumer{}
}
```

上述代码中的`kafka.MockConsumer`为我们模式Kafka消费者的一个实现，代码如下：

```go
package kafka
...
type MockConsumer struct {}

func (m *MockConsumer) Poll() *Records {
	records := &Records{}
	records.Items = append(records.Items, "i am mock consumer.")
	return records
}
```

测试代码如下：

```go
package test
...
func TestKafkaInputPipeline(t *testing.T) {
	config := pipeline.Config{
		Name: "pipeline2",
		Input: plugin.Config{
			PluginType: plugin.InputType,
			Name:       "kafka",
		},
		Filter: plugin.Config{
			PluginType: plugin.FilterType,
			Name:       "upper",
		},
		Output: plugin.Config{
			PluginType: plugin.OutputType,
			Name:       "console",
		},
	}
	p := pipeline.Of(config)
	p.Start()
	p.Exec()
	p.Stop()
}
// 运行结果
=== RUN   TestKafkaInputPipeline
Console output plugin started.
Upper filter plugin started.
Kafka input plugin started.
Pipeline started.
Output:
	Header:map[content:kafka], Body:[I AM MOCK CONSUMER.]
Kafka input plugin stopped.
Upper filter plugin stopped.
Console output plugin stopped.
Pipeline stopped.
--- PASS: TestKafkaInputPipeline (0.00s)
PASS
```

## 桥接模式（Bridge Pattern）

![桥接模式结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi00awfcxcj31f20l01ky.jpg)

### 简述

桥接模式主要用于**将抽象部分和实现部分进行解耦，使得它们能够各自往独立的方向变化**。它解决了在模块有多种变化方向的情况下，用继承所导致的类爆炸问题。举一个例子，一个产品有形状和颜色两个特征（变化方向），其中形状分为方形和圆形，颜色分为红色和蓝色。如果采用继承的设计方案，那么就需要新增4个产品子类：方形红色、圆形红色、方形蓝色、圆形红色。如果形状总共有m种变化，颜色有n种变化，那么就需要新增m*n个产品子类！现在我们使用桥接模式进行优化，将形状和颜色分别设计为一个抽象接口独立出来，这样需要新增2个形状子类：方形和圆形，以及2个颜色子类：红色和蓝色。同样，如果形状总共有m种变化，颜色有n种变化，总共只需要新增m+n个子类！

![桥接模式示例](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi01kwmua8j31hs0s47wj.jpg)

上述例子中，我们通过将形状和颜色抽象为一个接口，使产品不再依赖于具体的形状和颜色细节，从而达到了解耦的目的。**桥接模式本质上就是面向接口编程，可以给系统带来很好的灵活性和可扩展性**。如果一个对象存在多个变化的方向，而且每个变化方向都需要扩展，那么使用桥接模式进行设计那是再合适不过了。

### Go实现

回到消息处理系统的例子，一个`Pipeline`对象主要由`Input`、`Filter`、`Output`三类插件组成（**3个特征**），因为是插件化的系统，不可避免的就要求支持多种`Input`、`Filter`、`Output`的实现，并能够灵活组合（**有多个变化的方向**）。显然，`Pipeline`就非常适合使用桥接模式进行设计，实际上我们也这么做了。我们将`Input`、`Filter`、`Output`分别设计成一个抽象的接口，它们按照各自的方向去扩展。`Pipeline`只依赖的这3个抽象接口，并不感知具体实现的细节。

![使用桥接模式设计的Pipeline](https://tva1.sinaimg.cn/large/007S8ZIlgy1gi0i6xpj9sj318a0nk4qq.jpg)

```go
package plugin
...
type Input interface {
	Plugin
	Receive() *msg.Message
}

type Filter interface {
	Plugin
	Process(msg *msg.Message) *msg.Message
}

type Output interface {
	Plugin
	Send(msg *msg.Message)
}
```

```go
package pipeline
...
// 一个Pipeline由input、filter、output三个Plugin组成
type Pipeline struct {
	status plugin.Status
	input  plugin.Input
	filter plugin.Filter
	output plugin.Output
}
// 通过抽象接口来使用，看不到底层的实现细节
func (p *Pipeline) Exec() {
	msg := p.input.Receive()
	msg = p.filter.Process(msg)
	p.output.Send(msg)
}
```

测试代码如下：

```go
package test
...
func TestPipeline(t *testing.T) {
	p := pipeline.Of(pipeline.DefaultConfig())
	p.Start()
	p.Exec()
	p.Stop()
}
// 运行结果
=== RUN   TestPipeline
Console output plugin started.
Upper filter plugin started.
Hello input plugin started.
Pipeline started.
Output:
	Header:map[content:text], Body:[HELLO WORLD]
Hello input plugin stopped.
Upper filter plugin stopped.
Console output plugin stopped.
Pipeline stopped.
--- PASS: TestPipeline (0.00s)
PASS
```

## 总结

本文主要介绍了结构型模式中的组合模式、适配器模式和桥接模式。**组合模式**主要解决代码复用的问题，相比于继承关系，组合模式可以避免继承层次过深导致的代码复杂问题，因此面向对象设计领域流传着**组合优于继承**的原则，而Go语言的设计也很好实践了该原则；**适配器模式**可以看作是两个不兼容接口之间的桥梁，可以将一个接口转换成Client所希望的另外一个接口，解决了模块之间因为接口不兼容而无法一起工作的问题；**桥接模式**将模块的抽象部分和实现部分进行分离，让它们能够往各自的方向扩展，从而达到解耦的目的。

下一篇文章，我们将继续聚焦于结构型模式，介绍完剩余的4种结模式：装饰模式、外观模式、享元模式和代理模式。
