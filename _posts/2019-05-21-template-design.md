---
layout: post
title: Template Method
subtitle: 模板方法模式
date: 2019-05-21
categories: go
cover: 
tags: all 设计模式
---

> 库存 go-设计模式

> 参考[http://c.biancheng.net/view/1317.html](http://c.biancheng.net/view/1317.html)

模板方法用于当我们知道通用算法流程，不知道具体步骤算法的情况

使用抽象类定义好算法执行流程，将可变的算法延迟到子类实现

在golang中，由于不存在抽象类和真正的继承，所以只能通过一个基础类来充当抽象类

子类通过组合基础类来实现通用方法的继承

基础类通过一个接口指向子类(****这一步实际上代替了java中的抽象类多态**)

```go
package templateMethod

import (
	"fmt"
	"testing"
)

type ITemplate interface {
	templateMethod()
	specifyMethod()
	abstractMethod1()
	abstractMethod2()
}

type Base struct {
	ext ITemplate
}

func (b *Base) templateMethod()  {
	b.specifyMethod()
	b.abstractMethod1()
	b.abstractMethod2()
}

func (b *Base) specifyMethod()  {
	fmt.Println("执行通用算法")
}

func (b *Base) abstractMethod1()  {
	if b.ext == nil {
		return
	}
	b.ext.abstractMethod1()
}

func (b *Base) abstractMethod2()  {
	if b.ext == nil {
		return
	}
	b.ext.abstractMethod2()
}

type Ext struct {
	Base
}

func (b *Ext) abstractMethod1()  {
	fmt.Println("执行扩展1 算法1")
}

func (b *Ext) abstractMethod2()  {
	fmt.Println("执行扩展1 算法2")
}

type Ext2 struct {
	Base
}

func (b *Ext2) abstractMethod1()  {
	fmt.Println("执行扩展2 算法1")
}

func (b *Ext2) abstractMethod2()  {
	fmt.Println("执行扩展2 算法2")
}

func TestTemplate(t *testing.T)  {
	templateMethod := Base{}

	templateMethod.ext = &Ext{}
	templateMethod.templateMethod()
	
	templateMethod.ext = &Ext2{}
	templateMethod.templateMethod()

}
---------------------------------------------------------------
执行通用算法
执行扩展1 算法1
执行扩展1 算法2
执行通用算法
执行扩展2 算法1
执行扩展2 算法2
--- PASS: TestTemplate (0.00s)
```
<img src="/img/templateMethod1.png">

#### 附(uml)
```
@startuml
interface ITemplate {
	templateMethod()
	specifyMethod()
	abstractMethod1()
	abstractMethod2()
}
class Base {
	ext ITemplate

    templateMethod()
	specifyMethod()
	abstractMethod1()
	abstractMethod2()
}


class ext {
	Base

	abstractMethod1()
	abstractMethod2()
}


class ext2 {
	Base
    
	abstractMethod1()
	abstractMethod2()
}

class main

main ..> Base
ITemplate <|.. Base
ITemplate <|.. ext
ITemplate <|.. ext2
Base *- ext
Base *- ext2
Base ..>ITemplate

@enduml
```