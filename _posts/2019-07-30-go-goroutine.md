---
layout: post
title: Goroutine
subtitle: go 协程
date: 2019-07-30
categories: go
cover: 
tags: all go 多线程
---

> 读《Go语言高级编程》笔记

> 不要通过共享内存来通信，而应通过通信来共享内存。

### goroutine优势
goroutine最大的优势在于并发度极高，原因在于：
- 栈大小
	- 线程栈的常规实现是固定大小的（一般2M），大了浪费太多，小了容易栈溢出
	- go协程栈大小是动态调整的，初始化只有2k或4k，最大可以达到1G
- 调度器
	- 常规实现是直接采用系统线程，线程调度存在内核态与用户态的切换
	- go实现了自身的调度器，协程切换只发生在用户态，并且只有在阻塞时才会导致调度

### 初始化顺序
<img src="/img/goroutine-main-init.png">

Go程序的初始化和执行总是从main.main函数开始的。但是如果main包里导入了其它的包，则会按照顺序递归初始化导入包，创建和初始化这个包的常量和变量，之后执行包里的init函数，如果一个包有多个init函数的话，实现可能是以文件名的顺序调用，同一个文件内的多个init则是以出现的顺序依次调用。最终，在main包的所有包常量、包变量被创建和初始化，并且init函数被执行后，才会进入main.main函数，程序开始正常执行。

### 常见的并发模型
#### 生产者消费者

```go
// 生产者: 生成 factor 整数倍的序列
func Producer(factor int, out chan<- int) {
	for i := 0; ; i++ {
		out <- i*factor
		time.Sleep(time.Millisecond * 500)
	}
}

// 消费者
func Consumer(in <-chan int) {
	// 方式一
	//for v := range in {
	//	fmt.Println(v)
	//}

	// 方式二
	for  {
		select {
		case i := <- in:
			fmt.Println(i)
		}
	}
}

func main() {
	ch := make(chan int, 64) // 成果队列

	go Producer(3, ch) // 生成 3 的倍数的序列
	go Producer(5, ch) // 生成 5 的倍数的序列
	go Consumer(ch)    // 消费 生成的队列

	// Ctrl+C 退出
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
	fmt.Printf("quit (%v)\n", <-sig)
}

```
#### 发布-订阅
```go
type (
	subscriber chan interface{}        // 订阅者为一个管道
	topicFunc func(v interface{}) bool // 主题过滤器
)

// 发布者对象
type Publisher struct {
	m           sync.RWMutex             // 读写锁
	buffer      int                      // 订阅队列的缓存大小
	timeout     time.Duration            // 发布超时时间
	subscribers map[subscriber]topicFunc // 订阅者信息
}

// 构建一个发布者对象, 可以设置发布超时时间和缓存队列的长度
func NewPublisher(publishTimeout time.Duration, buffer int) *Publisher {
	return &Publisher{
		buffer:      buffer,
		timeout:     publishTimeout,
		subscribers: make(map[subscriber]topicFunc),
	}
}

// 添加一个新的订阅者，订阅全部主题
func (p *Publisher) Subscribe() subscriber {
	return p.SubscribeTopic(nil)
}

// 添加一个新的订阅者，订阅过滤器筛选后的主题
func (p *Publisher) SubscribeTopic(topic topicFunc) subscriber {
	ch := make(subscriber, p.buffer)
	p.m.Lock()
	p.subscribers[ch] = topic
	p.m.Unlock()
	return ch
}

// 退出订阅
func (p *Publisher) Evict(sub subscriber) {
	p.m.Lock()
	defer p.m.Unlock()

	delete(p.subscribers, sub)
	close(sub)
}

// 发布一个主题
func (p *Publisher) Publish(v interface{}) {
	p.m.RLock()
	defer p.m.RUnlock()

	var wg sync.WaitGroup
	for sub, topic := range p.subscribers {
		wg.Add(1)
		go p.sendTopic(sub, topic, v, &wg)
	}
	wg.Wait()
}

// 关闭发布者对象，同时关闭所有的订阅者管道。
func (p *Publisher) Close() {
	p.m.Lock()
	defer p.m.Unlock()

	for sub := range p.subscribers {
		delete(p.subscribers, sub)
		close(sub)
	}
}

// 发送主题，可以容忍一定的超时
func (p *Publisher) sendTopic(sub subscriber, topicFilter topicFunc, v interface{}, wg *sync.WaitGroup) {
	defer wg.Done()
	if topicFilter != nil && !topicFilter(v) {
		return
	}

	select {
	case sub <- v:
	case <-time.After(p.timeout):
	}
}

func TestPubSub(t *testing.T) {
	p := NewPublisher(100*time.Millisecond, 10)
	defer p.Close()

	all := p.Subscribe()
	golang := p.SubscribeTopic(func(v interface{}) bool {
		if s, ok := v.(string); ok {
			return strings.Contains(s, "golang")
		}
		return false
	})

	go func() {
		for msg := range all {
			fmt.Println("all:", msg)
		}
	}()

	go func() {
		for msg := range golang {
			fmt.Println("golang:", msg)
		}
	}()

	p.Publish("hello,  world!")
	p.Publish("hello, golang!")

	// 运行一定时间后退出
	time.Sleep(3 * time.Second)
}
```

- 订阅后返回一个接收管道
- 发布主题时遍历所有订阅者，通过主题过滤器决定是否向订阅者发送信息

#### 控制并发数
通过带缓存空间的channel进行并发控制
```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func() {
			limit <- 1
			w()
			<-limit
		}()
	}
	select{}
}
```

#### 赢者为王
有时候我们需要同时进行多个任务，但又只需要最快返回的一个结果
```go
func main() {
	ch := make(chan string, 32)

	go func() {
		ch <- searchByBing("golang")
	}()
	go func() {
		ch <- searchByGoogle("golang")
	}()
	go func() {
		ch <- searchByBaidu("golang")
	}()

	fmt.Println(<-ch)
}
```

#### 素数筛

<img src="/img/prime-sieve.png">

```go
// 返回生成自然数序列的管道: 2, 3, 4, 5, ...
func GenerateNatural() chan int {
	ch := make(chan int)
	go func() {
		for i := 2; ; i++ {
			ch <- i
		}
	}()
	return ch
}

// 管道过滤器: 删除能被素数（序列首位数）整除的数, 并生成下一级序列管道
func PrimeFilter(in <-chan int, prime int) chan int {
	out := make(chan int)
	go func() {
		for {
			if i := <-in; i%prime != 0 {
				out <- i
			}
		}
	}()
	return out
}

func TestPrime(t *testing.T) {
	ch := GenerateNatural() // 自然数序列: 2, 3, 4, ...
	for i := 0; i < 100; i++ {
		prime := <-ch // 新出现的素数（管道输出序列第一位即为确定的素数）
		fmt.Printf("%v: %v\n", i+1, prime)
		ch = PrimeFilter(ch, prime) // 基于新素数构造过滤器，返回新管道
	}
}
```
素数筛中每一个管道输出的一个值即为确定的素数，再基于此素数对序列进行过滤。

此例中将存在100个协程，并且协程调度频繁。

### 并发的安全退出
- 通过select 进行超时判断

```go
select {
case v := <-in:
	fmt.Println(v)
case <-time.After(time.Second):
	return // 超时
}
```
- 通过channel关闭通知goroutine退出，并使用waitGroup安全结束

```go
func worker(wg *sync.WaitGroup, cannel chan bool) {
	defer wg.Done()

	for {
		select {
		default:
			fmt.Println("hello")
		case <-cannel:
			return
		}
	}
}

func main() {
	cancel := make(chan bool)

	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(&wg, cancel)
	}

	time.Sleep(time.Second)
	close(cancel)
	wg.Wait()
}
```
- 通过context包

```go
func worker(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()

	for {
		select {
		default:
			fmt.Println("hello")
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(ctx, &wg)
	}

	time.Sleep(time.Second)
	cancel()

	wg.Wait()
}

```

### 附记录一个个人碰到的需求
- 一个客户端的连接请求需要启动一个goroutine定时任务
- 当连接断掉时，goroutine退出
- 若因未知原因（休眠期）任务未退出，而同一个客户端重新登录时，则需要关闭前一个goroutine

```go
var con = sync.Map{}


func runOne(key int, times int, ctx context.Context) {
	for { // 此处循环层数及等待时长为业务需求
		for i := 0; i < times; i++ {
			fmt.Println(key, i)
			// 执行任务，若失败，则退出
			select {
			case <-ctx.Done():
				return
			case <-time.After(time.Second):
			}
		}
		select {
		case <-ctx.Done():
			return
		case <-time.After(time.Second * 3):
		}
	}
}

func login(key int) {
	v, ok := con.Load(key)
	if ok {
		v.(context.CancelFunc)()
	}
	ctx, cancel := context.WithCancel(context.Background())
	con.Store(key, cancel)
	go runOne(key, 10, ctx)
}

func TestCloseOpen(t *testing.T) {
	login(10001)
	time.Sleep(time.Second * 14)
	login(10001)
	time.Sleep(time.Second * 5)
	login(10002)
	time.Sleep(time.Hour)
}
```