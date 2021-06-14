---
title: 如何高效编写Go单元测试（一）
date: 2020-12-22
categories:
 - Golang
tags:
 - golang
 - 单元测试
---

## 前言

**单元测试**是用来对一个模块、一个函数或者一个类来进行正确性检验的测试工作。我们根据它来检验代码的行为是否和预期的一样，如果单元测试不通过，要么代码有bug，要么测试条件输入不正确，总之，需要修复使单元测试能够通过。单元测试一个最大的好处，就是确保一个程序模块的行为符合我们设计的预期，在将来对代码进行修改/重构时，还能最大限度地保证代码的行为仍然正确。

Go对单元测试的支持已经相当友好了，原生的`go test`标准库就是专门用来进行单元测试的编写的。使用`go test`编写单元测试时需要遵循一些约定，比如所有测试代码都需要添加到`_test.go`结尾的测试文件中，这样在使用`go build`进行构建时，测试代码才会被排除在外。另外，每个测试函数都必须导入`testing`包，测试函数的名字必须以`Test`开头，且**跟在`Test`后面的后缀名必须以大写开头**，因此测试函数的声明应该是这样的：

```go
  func TestSin(t *testing.T) { /* ... */ }
  func TestCos(t *testing.T) { /* ... */ }
  func TestLog(t *testing.T) { /* ... */ }
```

`_test.go`测试文件通常和需要被测试的文件放在同一个包内，比如有如下的一段待测试的代码（在`word`包目录下的`word.go`文件）：

```go
package word
// 判断一个字符串s是否时回文字符串
func IsPalindrome(s string) bool {
    for i := range s {
        if s[i] != s[len(s)-1-i] {
            return false
        } 
    }
    return true
}
```

那么在编写测试时，我们同样在`word`包目录下，创建一个`word_test.go`文件，单元测试的代码如下：

```go
package word

import "testing"

func TestPalindrome(t *testing.T) {
  // "detartrated"是一个回文字符串，因此IsPalindrome("detartrated")的返回值应该为true
  // 如果返回false，则表示其实现有问题，需要使用t.Error进行错误报告
	if !IsPalindrome("detartrated") {
		t.Error(`IsPalindrome("detartrated") = false`)
	}
	if !IsPalindrome("kayak") {
		t.Error(`IsPalindrome("kayak") = false`)
	}
}

func TestNonPalindrome(t *testing.T) {
	if IsPalindrome("palindrome") {
    t.Error(`IsPalindrome("palindrome") = true`) 
  }
}
```

测试用例成功则打印：

```shell
=== RUN   TestPalindrome
--- PASS: TestPalindrome (0.00s)
PASS
```

用例执行失败则是：

```
=== RUN   TestNonPalindrome
--- FAIL: TestNonPalindrome (0.00s)
    nba_test.go:53: IsPalindrome("kayak") = true
FAIL
```

## 没有断言的go test框架

从单元测试的代码中可以看出，在判断`IsPalindrome`方法是否运行正确时，我们并未使用断言，而是通过`if`语句进行判断：如果结果错误则通过`t.Error`方法报告错误。这是因为Go原生就不支持断言，官网也给出了他们这样设计的原因，简单来说就是**不想让开发者在错误处理上进行偷懒**。

> Go doesn't provide assertions. They are undeniably convenient, but our experience has been that programmers use them as a crutch to avoid thinking about proper error handling and reporting...(详见https://golang.org/doc/faq#assertions)

但对单元测试来说，没有断言真的是很不方便。特别地，对以前做Java开发时习惯了使用`Junit`进行单元测试的同学而言更是难受。另外，使用`if`语句来对方法的输出结果进行判断，并进行对应的错误处理，会**导致单元测试代码充斥着大量的`if`分支，影响了代码的可读性**。

 *那么，有没有更好的解决方法呢？*

针对这个问题，我们可以引入第三方的断言库来解决。

## 使用testify进行断言

在第三方断言库的选择上，就活跃度和易用性而言，[testify](https://github.com/stretchr/testify)都是最佳的选择。

> testify：https://github.com/stretchr/testify

现在，我们使用testify为上述`IsPalindrome`的单元测试用例进行重构：

```go
package word

import "testing"
import "github.com/stretchr/testify/assert"

func TestPalindrome(t *testing.T) {
  // 断言IsPalindrome方法的返回值为True
	assert.True(t, IsPalindrome("detartrated"))
	assert.True(t, IsPalindrome("kayak"))
}

func TestNonPalindrome(t *testing.T) {
  // 断言IsPalindrome方法的返回值为False
	assert.False(t, IsPalindrome("palindrome"))
}
```

测试用例成功则打印：

```shell
=== RUN   TestPalindrome
--- PASS: TestPalindrome (0.00s)
PASS
```

用例执行失败则是：

```go
=== RUN   TestNonPalindrome
    palindrome_test.go:23: 
        	Error Trace:	palindrome_test.go:23
        	Error:      	Should be false
        	Test:       	TestNonPalindrome
--- FAIL: TestNonPalindrome (0.00s)
```

由此可见，重构后的测试用例更加简洁，可读性也更加好了。而且，在断言出错的情况下，打印出来的错误信息也相对丰富。

testify底层实现也是基于go test框架，在上述用法中，每个`assert`方法都将`testing.T`作为第一个入参，以此来使用go test框架的错误报告的基础能力。另外，如果你在一个测试方法中需要断言很多次，每次都传参`testing.T`就显得比较繁琐，那么可以这样实现来简化：

```go
func TestSomething(t *testing.T) {
  // 创建一个assert实例，只需传参testing.T一次
	assert := assert.New(t)

	// 断言两者相等
	assert.Equal(123, 123)

	// 断言两者不相等
	assert.NotEqual(123, 456)

	// 断言某个实例为nil
	assert.Nil(object)

	// 断言object实例不为nil
	if assert.NotNil(object) {

		// 当知道object不为nil之后，就可以安全地对其进行访问了
		// 进一步断言object的Value属性的值为"Something"
		assert.Equal("Something", object.Value)
	}
}
```

testify有着极其丰富的断言方法，除了上述几个`assert.True`、`assert.Nil`、`assert.Equal`常用的断言方法以外，还有用来断言目录是否存在的`assert.DirExists`；断言某个方法是否抛出`panic`的`assert.Panics`；断言字符串是否符合指定正则表达式的`assert.Regexp`，等等。总之，testify绝对能够满足你在Go单元测试断言方面的所有需要。

> 更多用法请参考testify assert API文档：https://godoc.org/github.com/stretchr/testify/assert

## 总结

本文主要介绍了如何使用testify库来为Go单元测试引入断言的能力，以此来使得测试用例代码更加简洁。`assert`只是testify库提供能力之一，比如它还有`mock`包，提供了“**打桩**”的能力。说起“**打桩**”，写过Java单元测试的同学一定很熟悉，Java中的**Mockito**和**PowerMock**框架就提供了类似的功能。本文之所以没有介绍testify库的mock功能，是因为还有更加好用的mock库。下一篇文章，我们将介绍如何高效、优雅地在Go单元测试中进行打桩。

