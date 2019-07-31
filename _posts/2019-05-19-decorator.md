---
layout: post
title: Decorator
subtitle: 装饰模式
date: 2019-07-22
categories: go
cover: 
tags: all go 设计模式
---

> 库存 go-设计模式

#### 第一种 类装饰

```go
package decorator

import (
	"fmt"
	"testing"
)

type IMedia interface {
	play()
}

type Mp3 struct {
	song string
}

func (m *Mp3) play() {
	fmt.Println("mp3 playing:", m.song)
}

type Mp4 struct {
	media IMedia
}

func (m *Mp4)play()  {
	fmt.Println("播放专辑图片")
	m.media.play()
	fmt.Println("other operate")
}

type Phone struct {
}

func (Phone) PlayMedia(m IMedia)  {
	m.play()
}

func TestDecorator(t *testing.T) {
	// 早期的手机只有音乐播放功能
	phoneOld := Phone{}
	phoneOld.PlayMedia(&Mp3{"以父之名"})

	//新手机播放音乐时播放专辑图片
	phoneNew := Phone{}
	phoneNew.PlayMedia(&Mp3{"夜的第七章"})
	phoneNew.PlayMedia(&Mp4{&Mp3{"夜的第七章"}})
}
```
这种实现与适配器有些类似，都是扩展新功能而不修改旧的类

**区别在于：**
- 适配器定义新的功能接口，通过适配器做转换，而装饰者是扩展，并不新增功能接口
- 适配器是通过旧的接口调用新的接口（**换**），而装饰者是新的实现在添加新的功能后调用旧的实现（**增**）

<img src="/img/decorator1.jpg">

#### 第二种 方法装饰
```go
package decorator

import (
	"fmt"
	"testing"
)

type handle func(s string)

func baseHandle(s string)  {
	fmt.Println("base handle:", s)
}

func Decorator(h handle) handle {
	return func(s string) {
		fmt.Println("decorator handle start")
		h(s)
		fmt.Println("decorator handle end")
	}
}

func TestDecoratorFun(t *testing.T)  {
	var handler handle
	handler = baseHandle
	handler("操作-----------1")

	handler = Decorator(baseHandle)
	handler("操作-----------2")
}
-------------------------------------------
=== RUN   TestDecoratorFun
base handle: 操作-----------1
decorator handle start
base handle: 操作-----------2
decorator handle end
--- PASS: TestDecoratorFun (0.00s)
PASS
```


#### 附(uml)
```
@startuml 
interface IMedia {
	play()
}

class Mp3 {
	song string

    play()
}

class Mp4 {
	media IMedia

    play()
}

class Phone {
    PlayMedia(m IMedia)
}

class main {
    
}

IMedia <|.. Mp3
IMedia <|.. Mp4
IMedia <.. Mp4
IMedia <.. Phone
Phone <.. main

@enduml
```