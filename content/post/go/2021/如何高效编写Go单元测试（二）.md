---
title: 如何高效编写Go单元测试（二）
date: 2021-01-10
categories:
 - Golang
tags:
 - golang
 - 单元测试
---

## 前言

上一篇文章《如何高效编写Go单元测试（一）》主要介绍了如何使用第三方断言库来使Go单元测试的代码更加简洁和具备可读性，本文我们来聊聊单元测试中的“**打桩**”。

```go
// 判断一个字符串s是否是回文字符串
func IsPalindrome(s string) bool {
    for i := range s {
        if s[i] != s[len(s)-1-i] {
            return false
        } 
    }
    return true
}
```

`go test`框架足以应对像上述`IsPalindrome`这种简单的方法的单元测试，但是对于一些较为复杂的方法和模块，`go test`多多少少会显得力不从心。比如当你需要对依赖了很多第三方库或者跟平台、环境强相关的模块进行单元测试时，仅仅使用`go test`往往不能使测试用例正常的执行结束，更别说达到代码验证的目的了。

```go
// 判断/opt/container/config.properties文件中是否包含str字符串
func isConfigFileContain(str string) bool {
	file, err := os.Open("/opt/container/config.properties")
	if err != nil {
		panic(err)
	}
	content, err := ioutil.ReadAll(file)
	if err != nil {
		panic(err)
	}
	return strings.Contains(string(content), str)
}
```

假设想要对上述`isConfigFileContain`函数进行单元测试，因为函数本身依赖了程序实际运行环境才有的`/opt/container/config.properties`文件，所以如果对其进行如下的单元测试，运行结果肯定出错。

```go
func TestIsConfigFileContain(t *testing.T) {
	ast := assert.New(t)
	ast.True(isConfigFileContain("test"))
}
```

运行结果如下：

```go
=== RUN   TestIsConfigFileContain
--- FAIL: TestIsConfigFileContain (0.00s)
panic: open /opt/container/config.properties: no such file or directory [recovered]
	panic: open /opt/container/config.properties: no such file or directory
```

果不其然，由于单元测试的环境并没有`/opt/container/config.properties`文件，因此用例执行失败。

解决该问题的思路有两种，（1）在运行单元测试用例之前先把`/opt/container/config.properties`文件创建好；（2）使用一种技术手段让测试用例在调用`os.Open("/opt/container/config.properties")`时能够按照我们的预想返回相应的结果。

*这两种思路都属于广义上的“打桩”，那么我们常说的单元测试中的“打桩”究竟是什么？它又是用来解决什么问题的呢？*

## 什么是“打桩”

在单元测试中，通常可以将所涉及的对象分为两种，主要测试对象和次要测试对象。比如在上述例子中，我们测试的主要测试对象就是`IsPalindrome`函数，而次要测试对象则是`os.Open`和`ioutil.ReadAll`函数。一般地，在测试用例中我们只需关注主要测试对象的行为是否正确。**对于次要测试对象，我们通常只会关注主要测试对象和次要测试对象之间的交互，比如是否被调用、调用参数、调用的次数、调用的结果等，至于次要测试对象是如何执行的，这些细节过程我们并不关注**。

因此，在进行单元测试中（特别是次要测试对象需要依赖特定的条件时，比如上述例子中依赖于`/opt/container/config.properties`文件的存在），我们常常选择使用一个**模拟对象**来替换次要测试对象，以此来模拟真实场景，对主要测试对象进行测试。而“*使用一个模拟对象来替换次要测试对象*”这个行为，我们通常称之为“**打桩**”。因此，**“打桩”的作用就是在单元测试中让我们从次要测试对象的繁琐依赖中解脱出来，进而能够聚焦于对主要测试对象的测试上**。

## stub与mock的区别

stub和mock是两种最常见的打桩手段，它们都能够用来替换次要测试对象，从而实现对一些复杂依赖的隔离，但是它们在实现和关注点上又有所区别。为了解它们之间的区别，考虑以下一段代码：

```go
// 邮箱消息
type Message struct {}

// 邮箱服务
type MailService interface {
  Start()
	Send(msg Message)
  Stop()
}

// 订单
type Order struct {
	mailService MailService
}

// 创建新订单
func NewOrder(service MailService) *Order {
	return &Order{mailService:service}
}

// 进行一些业务操作
func (o *Order) DoSomething() {
	...
}

// 关闭订单时，会发送邮件
func (o *Order) Close() {
  ...
  o.mailService.Send(Message{})
}

```

假设有一个订单业务`Order` ，它依赖了一个邮箱服务`MailService`。系统在创建订单之后，执行`DoSometing`方法进行一些业务操作，并在关闭订单时使用邮箱服务发送一个消息通知用户。现在我们要测试在订单关闭时，是否已经发送邮件。由于在单元测试中，一般不会使用真实的邮箱服务进行邮件发送，因此需要采用打桩技术来隔离邮箱服务的真实行为。下面，我们分别使用stub和mock来完成这一次的打桩。

如果使用stub来对`MailService`进行打桩，那么就要进行如下的实现：

```go
type StubMailService struct {
	messages []Message
}

func (s *StubMailService) Start() {
  // 空实现
}

func (s *StubMailService) Send(msg Message) {
	s.messages = append(s.messages, msg)
}

func (s *StubMailService) Stop() {
  // 空实现
}

func (s *StubMailService) SendNum() int {
	return len(s.messages)
}

func NewStubMailService() *StubMailService {
	return &StubMailService{messages:make([]Message, 0)}
}
```

对应的测试用例如下：

```go
func TestOrder_Stub(t *testing.T) {
	stub := NewStubMailService()
	order := NewOrder(stub)
	order.DoSomething()
	order.Close()
	assert.Equal(t, 1, stub.SendNum())
}
```

如果使用mock来对`MailService`进行打桩（这里我们采用的是go mock框架），则对应的测试用例如下：

```go
func TestOrder_Mock(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mock := NewMockMailService(ctrl)
	mock.EXPECT().Send(Message{}).Times(1)

	order := NewOrder(mock)
	order.DoSomething()
	order.Close()
}
```

从上述两个测试用例可以看出，在实现上，使用stub进行打桩时，我们需要自己实现`MailService`接口；而使用mock时，则直接利用框架的能力生成一个实现了`MailService`接口的mock对象。对比来看，**mock更加简单一些，只需关注主要测试方法，也就是所谓的“just enough”；而stub则要实现具体的逻辑，即使是不需要关注的次要测试方法，也需要给出空实现**。

Martin Fowler在[《Mocks Aren’t Stubs》](https://www.martinfowler.com/articles/mocksArentStubs.html)一文中将单元测试中的验证（verification）分为**state verification**和**behavior verification**两种，stub属于前者，mock则属于后者。正如它们的名称所描述，state verification关注的是测试对象的状态；behavior verification关注的是测试对象的行为。如上述例子，同样是验证`Order.DoSomething`的逻辑，stub通过记录发送过的消息数量来实现，也即是状态；mock则通过`mock.EXPECT().Send(Message{}).Times(1)`来验证`MailService.Send`的调用次数来实现，也即是行为。

## Go单元测试中的“打桩” 

### 为全局变量打桩

全局变量也经常在代码中用到，因为它具有全局性，如果在单元测试用例中对全局变量做了修改，则在用例结束时，需要将其恢复为原来的值，避免对其他用例造成影响，比如：

```go
func TestGlobalVal(t *testing.T) {
	val1 := global.Val1 // 步骤1：记住原来的值
	global.Val1 = 5     // 步骤2：赋予新的值
	...                 // 测试用例代码
	global.Val1 = val1  // 步骤3：恢复原来的值
}

```

如果测试用例涉及到的全局变量很多，这样就需要对每个全局变量都执行上述3个步骤，显得代码很不简洁：

```go
func TestGlobalVal(t *testing.T) {
	val1 := global.Val1
	val2 := global.Val2
	val3 := global.Val3
	global.Val1 = 5
	global.Val2 = 7
	global.Val3 = 6
	... // 测试用例代码
	global.Val1 = val1
	global.Val2 = val2
	global.Val3 = val3
}
```

针对全局变量的打桩，[gostub](https://github.com/prashantv/gostub)为我们提供了一个简洁的实现方式。

> gostub：https://github.com/prashantv/gostub

对上述例子，下面使用gostub对其进行优化：

```go
func TestGlobalVal(t *testing.T) {
	// 对全局变量进行打桩
	stub := gostub.Stub(&global.Val1, 5).
		Stub(&global.Val2, 7).
		Stub(&global.Val3, 6)
	... // 测试用例代码，这里使用这3个全局变量时的值分别为5，7，6
	// 对全局变量进行复原
	stub.Reset()
}
```

从上述例子来看，使用gostub为全局变量打桩使得代码更加简洁，可读性更好了。

除了为全局变量打桩，gostub也可以对函数/方法进行打桩，但是其对代码有一定的限制条件，因此易用性并不是十分好。下一节，我们将介绍一个更加好用的函数/方法打桩神器。

### 为函数/方法打桩

Go语言中，如果在函数声明时给它加上一个接收者，即将该函数附加到一种类型上，就变成了方法。比如，在前文中`func IsPalindrome(s string) bool`就是一个典型的函数声明；而`func (s *StubMailService) Send(msg Message)` 则是一个典型的方法声明。

在Go单元测试中，用来对函数和方法进行打桩的框架不少，但是就易用性而言，属[monkey patch](https://github.com/bouk/monkey)最好。

> monkey patch：https://github.com/bouk/monkey

如果直接对前言中的`isConfigFileContain`函数进行单元测试，因为没有`/opt/container/config.properties`文件，函数中的`os.Open`和`ioutil.ReadAll`都会执行失败。前文有提到，解决方法有两种，一种是创建`/opt/container/config.properties`文件；另一种，我们可以使用mokey patch实现。只要通过monkey patch对`os.Open`和`ioutil.ReadAll`进行打桩即可，测试代码如下：

```go
func TestIsConfigFileContain_MonkeyPatch(t *testing.T) {
	ast := assert.New(t)
	// 对os.Open进行打桩，固定返回(&os.File{}, nil)
	monkey.Patch(os.Open, func(name string) (*os.File, error) {
		return &os.File{}, nil
	})
	// 对ioutil.ReadAll进行打桩，固定返回([]byte("test for monkey patch"), nil)
	monkey.Path(ioutil.ReadAll, func(r io.Reader) ([]byte, error) {
		return []byte("test for monkey patch"), nil
	})
	// 测试用例结束时，解除打桩，避免影响其他用例
	defer monkey.UnpatchAll()
	ast.True(isConfigFileContain("test"))
}
```

monkey patch对函数进行打桩的API很简单，形式如下`monkey.Patch(<target function>, <replacement function>)`，需要注意的是，在用例结束之后，记得调用`monkey.UnpatchAll`来解除打桩，避免影响其他用例。

使用monkey patch对方法的打桩方法与函数的打桩方法稍微有点差异，考虑如下代码：

```go
// 从远端key-value数据库中获取记录
func GetRecord(key string) (string, error) {
	cli := db.CreateClient()
	var val string
	if err := cli.Get("key1", &val); err != nil {
		return "", err
	}
	return val, nil
}
```

其中`db.CreateClient`创建一个一个连接远端数据库的客户端`db.Client`，其定义如下：

```go
package db
...
// 数据库服务端远程代理，实现db.KvDb接口
type Client struct {
	// RPC客户端
	cli *rpc.Client
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
...
// 工厂方法，返回远程代理实例
func CreateClient() *Client {
	rpcCli, err := rpc.Dial("tcp", "192.168.21.34:8088")
	if err != nil {
		fmt.Printf("Create rpc client failed, error: %v.", err)
		return nil
	}
	return &Client{cli: rpcCli}
}
```

现在需要对`GetRecord`进行单元测试，因为在测试环境中并没有远端数据库实例，因此直接执行如下的测试用例，会运行失败：

```go
func TestGetRecord(t *testing.T) {
	val, err := GetRecord("key1")
  assert.Nil(t, err)
	assert.Equal(t, "value1", val)
}
// 测试结果:
=== RUN   TestGetRecord
Create rpc client failed, error: dial tcp 192.168.21.34:8088: connect: connection refused.
--- FAIL: TestGetRecord (0.00s)
```

现在我们使用monkey patch来解决该问题：

```go
func TestGetRecord_MonkeyPatch(t *testing.T) {
	var c *db.Client // 因为"Client.Get"方法的接收者是指针类型，所以这里必须声明为指针类型
	monkey.PatchInstanceMethod(reflect.TypeOf(c), "Get", func(_ *db.Client, reply *string) error {
		*reply = "value1"
		return nil
	})
	defer monkey.UnpatchAll()
	val, err := GetRecord("key1")
	assert.Nil(t, err)
	assert.Equal(t, "value1", val)
}
// 测试结果:
=== RUN   TestGetRecord
--- PASS: TestGetRecord (0.00s)
```

如代码所述，使用monkey patch为方法进行打桩的用法为`monkey.PatchInstanceMethod(<type>, <name>, <replacement>)`，其中`type`通过`reflect.TypeOf`获得，值得注意的是，`type`必须跟方法定义的接收者类型一致，如上述代码中，`Client.Get`方法的接收者是指针类型，因此`type`必须声明为`*Client`类型。

> monkey patch并不支持对包私有（首字母小写）的函数/方法进行打桩。因为设计者认为包私有的函数/方法是不稳定的，因此对它们进行打桩可能会导致测试用例代码的频繁更改，得不偿失。

### 为接口打桩

在单元测试中，我们也经常需要对接口`interface`进行打桩，比如上文所述的订单业务例子就对接口`MailService`进行了打桩。在该例子中，我们介绍了两种对接口打桩的方式。一是自己创建一个实现了`MailService`接口的桩对象stub，但是该方法有一定限制，当接口比较复杂或嵌套很多层时，该方式将会变得异常麻烦，考虑如下`ComplexItf`接口定义：

```go
type Itf1 interface {
	Func1()
	Func2()
}

type Itf2 interface {
	Func3(itf Itf1)
	Func4()
}

type Itf3 interface {
	Itf1
	Func5()
}

type ComplexItf interface {
	Itf2
	Itf3
	Func6()
	Func7()
	Func8(itf2 Itf2)
  Func9() Itf1
  ...
}
```

对于这么复杂的`ComplexItf`接口，光是其自身的方法都已经很多，更别提嵌套的`Itf2`和`Itf3`了。更好的是采用第二种打桩方法，也就是mock方式，我们通常使用[gomock](https://github.com/golang/mock)框架来实现。

> gomock：https://github.com/golang/mock

使用gomock进行打桩时，我们完全将实现接口这一繁琐的工作交给了框架，比如使用gomock对`MailService`接口打桩时，在安装好gomock之后，只需执行如下命令，mock对象就会自动生成：

```shell
 mockgen -source=yrunz/mail_service.go -destination=yrunz/mock_mail_service.go -package=yrunz
```

其中`-source`表示`MailService`接口所在文件路径，`-destination`表示生成的mock对象所做的文件路径，`-package`表示mock对象的包名。生成的mock对象对应的代码如下：

```go
// Code generated by MockGen. DO NOT EDIT.
// Source: yrunz/mail_service.go

// Package yrunz is a generated GoMock package.
package yrunz

import (
	gomock "github.com/golang/mock/gomock"
	reflect "reflect"
)

// MockMailService is a mock of MailService interface
type MockMailService struct {
	ctrl     *gomock.Controller
	recorder *MockMailServiceMockRecorder
}

// MockMailServiceMockRecorder is the mock recorder for MockMailService
type MockMailServiceMockRecorder struct {
	mock *MockMailService
}

// NewMockMailService creates a new mock instance
func NewMockMailService(ctrl *gomock.Controller) *MockMailService {
	mock := &MockMailService{ctrl: ctrl}
	mock.recorder = &MockMailServiceMockRecorder{mock}
	return mock
}

// EXPECT returns an object that allows the caller to indicate expected use
func (m *MockMailService) EXPECT() *MockMailServiceMockRecorder {
	return m.recorder
}

// Start mocks base method
func (m *MockMailService) Start() {
	m.ctrl.T.Helper()
	m.ctrl.Call(m, "Start")
}

...
```

生成之后就可以使用`NewMockMailService`工厂方法创建一个mock对象进行单元测试了：

```go
func TestOrder_Mock(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()
	mock := NewMockMailService(ctrl)
	mock.EXPECT().Send(Message{}).Times(1)
	order := NewOrder(mock)
	order.DoSomething()
	order.Close()
}
```

我们注意到，mockgen工具生成的mock对象都是`Mock+接口名`的命名形式，并提供了一个`New+Mock+接口名`命名方式的工厂方法用于创建mock对象，其中工厂方法的入参为`*gomock.Controller`类型。

前文有提到，mock属于**behavior verification**，也就是行为验证方式。实际使用gomock框架时，我们通过调用mock对象的`EXPECT`设定预期返回值、预期动作以及调用次数验证等。比如上述例子中的`mock.EXPECT().Send(Message{}).Times(1)`表示mock对象的`Send`方法在测试用例中应该被调用一次，且入参为`Message{}`，否则用例验证失败。

gomock的一些常用的行为验证方法如下：

> .Do()：声明在匹配时要运行的操作。
>
> .DoAndReturn()：声明在匹配调用时要运行的操作，并模拟返回该函数的返回值。
>
> .Return()：在匹配调用时模拟返回该函数的返回值。
>
> .MaxTimes()：设置最大的调用次数
>
> .MinTimes()：设置最小的调用次数
>
> .AnyTimes()：运行调用次数为0或更多次。
>
> .Times()：设置调用次数为n次。

如果调用方法的入参不固定，可以使用gomock.Any()进行匹配，比如前面的例子中，可以这样：

```go
mock.EXPECT().Send(gomock.Any()).Times(1)
```

常用的参数匹配如下：

> gomock.Any()：匹配任意值
>
> gomock.Eq()：通过反射匹配到指定的类型
>
> gomock.Nil()：匹配nil

## 总结

本文主要主要介绍了单元测试中的“打桩”，**打桩主要作用是在单元测试中让我们从次要测试对象的繁琐依赖中解脱出来，进而能够聚焦于对主要测试对象的测试上**。常见的打桩技术stub和mock也并非完全相同，前者属于state verification，后者属于behavior verification。另外本文还介绍了Go单元测试的几种打桩框架，为全局变量打桩的**gostub**框架、为函数/方法打桩的**monkey patch**框架、为接口打桩的**go mock**框架。熟练地使用这些框架，能够让我们更加高效、优雅地编写Go单元测试。
