# Golang之内存泄漏

### 内存泄漏

内存泄漏说白了就是分配的内存\(或者变量\)不再使用，但是并没有被gc回收，而是继续占用内存。在Golang中内存泄漏大部分都跟channel的不正确使用有关。泄漏的原因是goroutine操作channel后，处于发送阻塞或者接收阻塞状态，而channel处于满或者空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致goroutine会处于一个一直等待的队列中，不见天日。

### 场景分析

##### 案例1: channel没有接收者

下面这段程序展示了并发地向3个站点请求资源，接收者只接收最快返回的那个。

```go
func goroutineLeak1() string {
	responses := make(chan string) //正确姿势：responses := make(chan string, 3)
  //3个发送者
	go func {responses <- request("asia.gopl.io")}()
	go func {responses <- request("europe.gopl.io")}()
	go func {responses <- request("americas.gopl.io")}()
  //1个接收者
	return <- responses
}
```

goroutineLeak1能正确地接收到最快返回的站点资源，但两个慢的goroutien会因为没有人接收而永远卡住，造成这两个goroutine永远没法被回收。正确的姿势是创建一个带3个buffer的channel，可以回收两个返回比较慢的goroutine。

##### 案例2: channel没有发送者

下面这段程序模拟生产者与消费者。

```go
func goroutineLeak2() {
	ch := make(chan int)
	//10接收者
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Printf("%d号消费者 接收到消息：%d", i, <-ch)
		}(i)
	}

	//1个发送者
	go func() {  //正确姿势: 需要生产10个消息
		ch <- rand.Int()
	}()
}
```

goroutineLeak2中创建10个goroutine来接收消息，但只有一个生产者生产消息，接收者goroutine会因为没有消费消息而阻塞，也没法被回收。正确的姿势是生产者创建10条消息，使得接收者能够完成消费。

##### 案例3: timer使用错误

time.After每次都会创建一个定时器，定时器未到触发时间，该定时器不会被gc回收，从而导致临时性的内存泄漏。

```go
func goroutineLeak3() {
	ch := make(chan string)
	//接收数据
	go func() {
		for {
			select {
			case <-time.After(3 * time.Second):
				fmt.Println("request timeout")
			case data := <-ch:
				fmt.Println("收到数据: ", data)
			}
		}
	}()

	//模拟发送数据
	for i := 0; i < 10000; i++ {
		ch <- "ok"
		time.Sleep(time.Second)
	}
}
```

正确姿势是创建timer定时器，每次需要启动定时器的时候，使用Reset方法重置定时器，这样就不用每次都要创建新的定时器了。

```go
func goroutineLeak3() {
	ch := make(chan string)
	go func() {
		timer := time.NewTimer(3 * time.Second)
		defer timer.Stop()
		for {
			timer.Reset(3 * time.Second)
			select {
			case <-timer.C:
				fmt.Println("request timeout")
			case data := <-ch:
				fmt.Println("收到数据: ", data)
			}
		}
	}()

	for i := 0; i < 10000; i++ {
		ch <- "ok"
		time.Sleep(time.Second)
	}
}
```

