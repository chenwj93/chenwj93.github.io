---
layout: post
title: go-kit essay
subtitle: go-kit 随记
date: 2019-07-26
categories: go
cover: 
tags: go go-kit 框架 微服务 中间件
---

> 库存 go-kit中间件学习
### middleware与serverOption
#### 1. 
```	
middleware 是包裹在endpoint外层的，是属于middleware的
serverOption 是加在server调用链中的
```
#### 2.
```
middleware 可以在endpoint前后都增加动作
serverOption 只能在执行前或执行后
```
#### 3.
```
middleware 更靠近业务层
serverOption 更靠近框架层
```

![image](https://wx3.sinaimg.cn/mw1024/007MotX5ly1g2v5f365omj30ho06w747.jpg)

### 限流器
设置一段时间内最多访问次数
> package ratelimit
```go
endpoint = MakeEndpoint(svc)
 // 超限报错
sumEndpoint = ratelimit.NewErroringLimiter(rate.NewLimiter(rate.Every(time.Second), 1))(endpoint)

 // 超限延迟
sumEndpoint = ratelimit.NewDelayingLimiter(rate.NewLimiter(rate.Every(time.Second), 1))(endpoint)
```

### 断路器
当服务panic次数过多时，断路器会阻止服务调用，直接返回错误
#### gobreaker
> package circuitbreaker
```go
endpoint = MakeSumEndpoint(svc)
circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name: ""
    MaxRequests: 1
    Interval: 0
    Timeout: 30
}))(endpoint)
```
```go
type Settings struct {
	Name          string
	MaxRequests   uint32
	Interval      time.Duration
	Timeout       time.Duration
	ReadyToTrip   func(counts Counts) bool
	OnStateChange func(name string, from State, to State)
}
```
- Name 断路器名称
- MaxRequests half-open状态期最大允许放行请求数
- Interval  closed状态时，重置计数的时间周期；如果配为0，那么在closed状态时不清理计数
- Timeout 切入open状态后切回half-open的时间；默认为60s，不可为0
- ReadyToTrip 函数决定进入Open状态的条件。在fail计数发生后，调用一次。如返回true,则切入open

**closed->open->half-open**

#### hystrix
> package circuitbreaker
```go
    endpoint = MakeEndpoint(svc)
	hystrix.ConfigureCommand("my-endpoint",   
        hystrix.CommandConfig{
            ErrorPercentThreshold: 5,
		    MaxConcurrentRequests: 2,
		    Timeout: 2000,
		    RequestVolumeThreshold: 5,
		    SleepWindow: 5000,
        })
    circuitbreaker.Hystrix("my-endpoint")(endpoint)
```
```go
type CommandConfig struct {
	Timeout                int `json:"timeout"`
	MaxConcurrentRequests  int `json:"max_concurrent_requests"`
	RequestVolumeThreshold int `json:"request_volume_threshold"`
	SleepWindow            int `json:"sleep_window"`
	ErrorPercentThreshold  int `json:"error_percent_threshold"`
}
```
- Timeout  请求超时时间 （毫秒）
- MaxConcurrentRequests  同一时间最大请求数（并发限流，**与限流器有区别**）
- RequestVolumeThreshold  达到这个阈值才进行错误率计算
- SleepWindow  熔断器开启多长时间后进入半开状态，测试服务是否恢复（放行一部分流量）（毫秒），失败后再次开启
- ErrorPercentThreshold 错误率超过这个值将开启熔断器

#### handybreaker
> package circuitbreaker
```go
    endpoint = MakeEndpoint(svc)
	breaker := breaker.NewBreaker(0.05) // 错误率阈值
	breaker.WithMinObservation(5) // 最小观察数
	breaker.WithWindow(time.Second * 5) // 计算断路阈值时的窗口时间
	breaker.WithCooldown(time.Second * 5) // 断路器开启后重试时间
    circuitbreaker.HandyBreaker(breaker)(endpoint)
```
**这个包还是不要用了**,breaker设置的参数都没有用，看源码
```go
type breaker struct {
	force   chan states
	allow   chan bool
	success chan time.Duration
	failure chan time.Duration

	config breakerConfig // 注意点
}
```
在这个结构体中，config类型不是指针，而handybreaker在NewBreaker时就启动了断路器监控，后面的参数设置由于不是指针类型导致传不进监控线程。

#### 比较
##### 特色：
> gobreaker :**半开限流**、**自定义状态改变函数**

> hystrix ：**超时控制**、**并发限流（限流器是一段时间控制）**、

> handybreaker：**目前不可修改配置**

### service discovery
```go
func Register(consulHost, consulPort, svcHost, svcPort string, logger log.Logger) (registar sd.Registrar) {

	// 创建Consul客户端连接
	var client consul.Client
	{
		consulCfg := api.DefaultConfig()
		consulCfg.Address = consulHost + ":" + consulPort
		consulClient, err := api.NewClient(consulCfg)
		if err != nil {
			logger.Log("create consul client error:", err)
			os.Exit(1)
		}

		client = consul.NewClient(consulClient)
	}

	// 设置Consul对服务健康检查的参数
	check := api.AgentServiceCheck{
		HTTP:     "http://" + svcHost + ":" + svcPort + "/health",
		Interval: "10s",
		Timeout:  "1s",
		Notes:    "Consul check service health status.",
	}

	port, _ := strconv.Atoi(svcPort)

	//设置微服务想Consul的注册信息
	reg := api.AgentServiceRegistration{
		ID:      "arithmetic" + uuid.New(),
		Name:    "arithmetic",
		Address: svcHost,
		Port:    port,
		Tags:    []string{"arithmetic", "raysonxin"},
		Check:   &check,
	}

	// 执行注册
	registar = consul.NewRegistrar(client, &reg, logger)
	return
}
```