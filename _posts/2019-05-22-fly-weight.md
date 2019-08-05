---
layout: post
title: Fly Weight
subtitle: 享元模式
date: 2019-05-22
categories: go
cover: 
tags: all go 设计模式
---

> 库存 go-设计模式

> 参考[http://c.biancheng.net/view/1317.html](http://c.biancheng.net/view/1317.html)

享元模式目的主要是为了共享对象，减少对象的创建

原理是区分不变与变化的维度

例如 高铁播报中每一条高铁线若干条高铁起始与终点站一样，状态不一样
```
杭州到北京的高铁即将出发
杭州到北京的高铁即将进站
杭州到北京的高铁正在检票
上海到杭州的高铁即将进站
```
那么每一条高铁线就是共有部分，状态则是执行时不同的选择

```go
package flyweight

import (
	"fmt"
	"testing"
	"time"
)

type IFlyWeight interface {
	operate(status string, s int)
}

type Aircraft struct {
	srcStation string
	desStation string
}

func (a *Aircraft)operate(status string, s int)  {
	time.Sleep(time.Second * time.Duration(s))
	fmt.Printf("the aircraft that from %s to %s is %s\n", a.srcStation, a.desStation, status)
}


var aircraft = map[string]IFlyWeight{}


func GetAircraft(src, des string) IFlyWeight {
	if air, ok := aircraft[src + des]; ok {
		fmt.Printf("aircraft that from %s to %s had exsit\n", src, des)
		return air
	} else {
		fmt.Printf("create aircraft from %s to %s\n", src, des)
		air := &Aircraft{src, des}
		aircraft[src + des] = air
		return air
	}
}

func TestFlyWeight(t *testing.T) {
	air1 := GetAircraft("杭州", "北京")
	air2 := GetAircraft("上海", "杭州")
	air3 := GetAircraft("杭州", "北京")
	go air1.operate("即将进站", 1)
	go air2.operate("正在检票", 0)
	go air3.operate("即将出发", 0)
	time.Sleep(time.Second * 3)
}
------------------------------------------------------------
create aircraft from 杭州 to 北京
create aircraft from 上海 to 杭州
aircraft that from 杭州 to 北京 had exsit
the aircraft that from 上海 to 杭州 is 正在检票
the aircraft that from 杭州 to 北京 is 即将出发
the aircraft that from 杭州 to 北京 is 即将进站
--- PASS: TestFlyWeight (3.00s)
```

**享元模式最终的执行条件或参数一定不能设置到对象中去，否则将有并发问题**

当然，在执行时将参数设置到对象中也就不能称其为享元模式了

享元模式和单例模式的区别在于单例模式确定只能生成一个对象，是在类中自身实现的，享元模式是通过工厂类实现对象复用，通过直接创建类方式也可以创建多个对象

<img src="/img/flyweight1.png">

#### 附(uml)
```
@startuml
interface IFlyWeight {
	operate(status string, s int)
}

class Aircraft {
	srcStation string
	desStation string

    operate(status string, s int)
}

class AircraftFactory {
    aircraft map[string]IFlyWeight{}

    GetAircraft(src, des string) IFlyWeight
}

AircraftFactory ..> IFlyWeight
IFlyWeight <|.. Aircraft

@enduml
```