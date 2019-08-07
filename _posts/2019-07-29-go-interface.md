---
layout: post
title: Go Interface
subtitle: 接口
date: 2019-07-29
categories: go
cover: 
tags: all 接口
---

> 读《Go语言高级编程》笔记

go中接口是对其它类型行为的抽象和概括。

与其它语言一个极大的不同在于go采用鸭子类型，即：只要一个类型实现了一个接口所有的方法，即可认为该类型是该接口的实现

```go
type I interface {
 	func1()
 	func2()
 	func3()
}

type I2 interface {
 	func1()
 	func2()
 	func4()
}

type S struct{

}

func (s *S) func1() {
 	println("func1")
}
func (s *S) func2() {
 	println("func2")
}
func (s *S) func3() {
 	println("func3")
}
func (s *S) func4() {
 	println("func4")
}

func main(){
 	var i I
 	var i2 I2
 	i = new(S)
 	i.func1()

 	i2 = new(S)
 	i2.func4()
}
```

**go中函数类型同样可以实现接口**
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

有时候对象和接口之间太灵活了，导致我们需要人为地限制这种无意之间的适配。常见的做法是定义一个含特殊方法来区分接口。比如runtime包中的Error接口就定义了一个特有的RuntimeError方法
```go
type runtime.Error interface {
	error

	// RuntimeError is a no-op function but
	// serves to distinguish types that are run time
	// errors from ordinary errors: a type is a
	// run time error if it has a RuntimeError method.
	RuntimeError()
}
```

另一种实现接口的方法是嵌入匿名接口，并且此种方式可以**不实现所有的接口方法**
```go
type ITest interface{
	testPrint1()
	testPrint2()
	testPrint3()
}

type PrintStruct struct {
	ITest
}

func (*PrintStruct) testPrint1() {
	fmt.Println("print 1")
}

func TestInterfaceImplement(t *testing.T) {
	var i ITest = new(PrintStruct)
	i.testPrint1()
}
```
**注意：**此种方式的弊端很明显，必须完全清楚该类型的实现，否则极其容易调到未实现的方法。

