# 九股文笔记-Golang之内存泄漏

#### 内存泄漏

内存泄漏说白了就是分配的内存\(或者变量\)不再使用，但是并没有被gc回收，而是继续占用内存。在Golang中内存泄漏大部分都跟channel的不正确使用有关。

#### 场景分析

##### 案例1: channel没有接收者

下面这段程序展示了并发地向3个站点请求资源，接收者只接收最快返回的那个。

```
func goroutineLeak1() string {
	responses := make(chan string) //正确姿势：responses := make(chan string, 3)
	go func {responses := <- request("asia.gopl.io")}()
	go func {responses := <- request("europe.gopl.io")}()
	go func {responses := <- request("americas.gopl.io")}()
	return <- responses
}
```

goroutineLeak1能正确地接收到最快返回的站点资源，但两个慢的goroutien会因为没有人接收而永远卡住，造成这两个goroutine永远没法被回收。正确的姿势是创建一个带3个buffer的channel，可以回收两个慢的goroutine。

##### 案例2: channel没有发送者

下面这段程序模拟生产者与消费者。

```
func goroutineLeak2() {
	ch := make(chan int) //正确姿势1: ch := make(chan int, 10)
	//10个生产者
	for i := 0; i < 10; i++ {
		go func() {
			<-ch
		}()
	}

	//1个消费者 
	go func() {  //正确姿势2: 起10个消费者goroutine 
		fmt.Println("消费者消费消息：", <-ch) 
	}
}
```

生产者通过10个goutine生产10个消息，但却消费者只能消费1个，生产者goroutine会因为没有消费者消费消息而阻塞，也没法被回收。正确的姿势是使用带10个buffer的channel或者创建10个消费者去消费消息。

##### 案例3: timer使用错误

time.After每次都会创建一个定时器，定时器未到触发时间，该定时器不会被gc回收，从而导致临时性的内存泄漏。

```
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

```
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

%23%23%23%23%20%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%0A%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E8%AF%B4%E7%99%BD%E4%BA%86%E5%B0%B1%E6%98%AF%E5%88%86%E9%85%8D%E7%9A%84%E5%86%85%E5%AD%98\(%E6%88%96%E8%80%85%E5%8F%98%E9%87%8F\)%E4%B8%8D%E5%86%8D%E4%BD%BF%E7%94%A8%EF%BC%8C%E4%BD%86%E6%98%AF%E5%B9%B6%E6%B2%A1%E6%9C%89%E8%A2%ABgc%E5%9B%9E%E6%94%B6%EF%BC%8C%E8%80%8C%E6%98%AF%E7%BB%A7%E7%BB%AD%E5%8D%A0%E7%94%A8%E5%86%85%E5%AD%98%E3%80%82%E5%9C%A8Golang%E4%B8%AD%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E5%A4%A7%E9%83%A8%E5%88%86%E9%83%BD%E8%B7%9Fchannel%E7%9A%84%E4%B8%8D%E6%AD%A3%E7%A1%AE%E4%BD%BF%E7%94%A8%E6%9C%89%E5%85%B3%E3%80%82%0A%0A%23%23%23%23%20%E5%9C%BA%E6%99%AF%E5%88%86%E6%9E%90%0A%23%23%23%23%23%20%E6%A1%88%E4%BE%8B1%3A%20channel%E6%B2%A1%E6%9C%89%E6%8E%A5%E6%94%B6%E8%80%85%0A%E4%B8%8B%E9%9D%A2%E8%BF%99%E6%AE%B5%E7%A8%8B%E5%BA%8F%E5%B1%95%E7%A4%BA%E4%BA%86%E5%B9%B6%E5%8F%91%E5%9C%B0%E5%90%913%E4%B8%AA%E7%AB%99%E7%82%B9%E8%AF%B7%E6%B1%82%E8%B5%84%E6%BA%90%EF%BC%8C%E6%8E%A5%E6%94%B6%E8%80%85%E5%8F%AA%E6%8E%A5%E6%94%B6%E6%9C%80%E5%BF%AB%E8%BF%94%E5%9B%9E%E7%9A%84%E9%82%A3%E4%B8%AA%E3%80%82%0A%60%60%60%0Afunc%20goroutineLeak1\(\)%20string%20%7B%0A%09responses%20%3A%3D%20make\(chan%20string\)%20%2F%2F%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF%EF%BC%9Aresponses%20%3A%3D%20make\(chan%20string%2C%203\)%0A%09go%20func%20%7Bresponses%20%3A%3D%20%3C\-%20request\(%22asia.gopl.io%22\)%7D\(\)%0A%09go%20func%20%7Bresponses%20%3A%3D%20%3C\-%20request\(%22europe.gopl.io%22\)%7D\(\)%0A%09go%20func%20%7Bresponses%20%3A%3D%20%3C\-%20request\(%22americas.gopl.io%22\)%7D\(\)%0A%09return%20%3C\-%20responses%0A%7D%0A%60%60%60%0AgoroutineLeak1%E8%83%BD%E6%AD%A3%E7%A1%AE%E5%9C%B0%E6%8E%A5%E6%94%B6%E5%88%B0%E6%9C%80%E5%BF%AB%E8%BF%94%E5%9B%9E%E7%9A%84%E7%AB%99%E7%82%B9%E8%B5%84%E6%BA%90%EF%BC%8C%E4%BD%86%E4%B8%A4%E4%B8%AA%E6%85%A2%E7%9A%84goroutien%E4%BC%9A%E5%9B%A0%E4%B8%BA%E6%B2%A1%E6%9C%89%E4%BA%BA%E6%8E%A5%E6%94%B6%E8%80%8C%E6%B0%B8%E8%BF%9C%E5%8D%A1%E4%BD%8F%EF%BC%8C%E9%80%A0%E6%88%90%E8%BF%99%E4%B8%A4%E4%B8%AAgoroutine%E6%B0%B8%E8%BF%9C%E6%B2%A1%E6%B3%95%E8%A2%AB%E5%9B%9E%E6%94%B6%E3%80%82%E6%AD%A3%E7%A1%AE%E7%9A%84%E5%A7%BF%E5%8A%BF%E6%98%AF%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E5%B8%A63%E4%B8%AAbuffer%E7%9A%84channel%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%9B%9E%E6%94%B6%E4%B8%A4%E4%B8%AA%E6%85%A2%E7%9A%84goroutine%E3%80%82%0A%0A%23%23%23%23%23%20%E6%A1%88%E4%BE%8B2%3A%20channel%E6%B2%A1%E6%9C%89%E5%8F%91%E9%80%81%E8%80%85%0A%E4%B8%8B%E9%9D%A2%E8%BF%99%E6%AE%B5%E7%A8%8B%E5%BA%8F%E6%A8%A1%E6%8B%9F%E7%94%9F%E4%BA%A7%E8%80%85%E4%B8%8E%E6%B6%88%E8%B4%B9%E8%80%85%E3%80%82%0A%60%60%60%0Afunc%20goroutineLeak2\(\)%20%7B%0A%09ch%20%3A%3D%20make\(chan%20int\)%20%2F%2F%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF1%3A%20ch%20%3A%3D%20make\(chan%20int%2C%2010\)%0A%09%2F%2F10%E4%B8%AA%E7%94%9F%E4%BA%A7%E8%80%85%0A%09for%20i%20%3A%3D%200%3B%20i%20%3C%2010%3B%20i%2B%2B%20%7B%0A%09%09go%20func\(\)%20%7B%0A%09%09%09%3C\-ch%0A%09%09%7D\(\)%0A%09%7D%0A%0A%09%2F%2F1%E4%B8%AA%E6%B6%88%E8%B4%B9%E8%80%85%20%0A%09go%20func\(\)%20%7B%20%20%2F%2F%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF2%3A%20%E8%B5%B710%E4%B8%AA%E6%B6%88%E8%B4%B9%E8%80%85goroutine%20%0A%09%09fmt.Println\(%22%E6%B6%88%E8%B4%B9%E8%80%85%E6%B6%88%E8%B4%B9%E6%B6%88%E6%81%AF%EF%BC%9A%22%2C%20%3C\-ch\)%20%0A%09%7D%0A%7D%0A%60%60%60%0A%E7%94%9F%E4%BA%A7%E8%80%85%E9%80%9A%E8%BF%8710%E4%B8%AAgoutine%E7%94%9F%E4%BA%A710%E4%B8%AA%E6%B6%88%E6%81%AF%EF%BC%8C%E4%BD%86%E5%8D%B4%E6%B6%88%E8%B4%B9%E8%80%85%E5%8F%AA%E8%83%BD%E6%B6%88%E8%B4%B91%E4%B8%AA%EF%BC%8C%E7%94%9F%E4%BA%A7%E8%80%85goroutine%E4%BC%9A%E5%9B%A0%E4%B8%BA%E6%B2%A1%E6%9C%89%E6%B6%88%E8%B4%B9%E8%80%85%E6%B6%88%E8%B4%B9%E6%B6%88%E6%81%AF%E8%80%8C%E9%98%BB%E5%A1%9E%EF%BC%8C%E4%B9%9F%E6%B2%A1%E6%B3%95%E8%A2%AB%E5%9B%9E%E6%94%B6%E3%80%82%E6%AD%A3%E7%A1%AE%E7%9A%84%E5%A7%BF%E5%8A%BF%E6%98%AF%E4%BD%BF%E7%94%A8%E5%B8%A610%E4%B8%AAbuffer%E7%9A%84channel%E6%88%96%E8%80%85%E5%88%9B%E5%BB%BA10%E4%B8%AA%E6%B6%88%E8%B4%B9%E8%80%85%E5%8E%BB%E6%B6%88%E8%B4%B9%E6%B6%88%E6%81%AF%E3%80%82%0A%0A%0A%23%23%23%23%23%20%E6%A1%88%E4%BE%8B3%3A%20timer%E4%BD%BF%E7%94%A8%E9%94%99%E8%AF%AF%0Atime.After%E6%AF%8F%E6%AC%A1%E9%83%BD%E4%BC%9A%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E5%AE%9A%E6%97%B6%E5%99%A8%EF%BC%8C%E5%AE%9A%E6%97%B6%E5%99%A8%E6%9C%AA%E5%88%B0%E8%A7%A6%E5%8F%91%E6%97%B6%E9%97%B4%EF%BC%8C%E8%AF%A5%E5%AE%9A%E6%97%B6%E5%99%A8%E4%B8%8D%E4%BC%9A%E8%A2%ABgc%E5%9B%9E%E6%94%B6%EF%BC%8C%E4%BB%8E%E8%80%8C%E5%AF%BC%E8%87%B4%E4%B8%B4%E6%97%B6%E6%80%A7%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E3%80%82%0A%60%60%60%0Afunc%20goroutineLeak3\(\)%20%7B%0A%09ch%20%3A%3D%20make\(chan%20string\)%0A%09%2F%2F%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE%0A%09go%20func\(\)%20%7B%0A%09%09for%20%7B%0A%09%09%09select%20%7B%0A%09%09%09case%20%3C\-time.After\(3%20\*%20time.Second\)%3A%0A%09%09%09%09fmt.Println\(%22request%20timeout%22\)%0A%09%09%09case%20data%20%3A%3D%20%3C\-ch%3A%0A%09%09%09%09fmt.Println\(%22%E6%94%B6%E5%88%B0%E6%95%B0%E6%8D%AE%3A%20%22%2C%20data\)%0A%09%09%09%7D%0A%09%09%7D%0A%09%7D\(\)%0A%0A%0A%09%2F%2F%E6%A8%A1%E6%8B%9F%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%0A%09for%20i%20%3A%3D%200%3B%20i%20%3C%2010000%3B%20i%2B%2B%20%7B%0A%09%09ch%20%3C\-%20%22ok%22%0A%09%09time.Sleep\(time.Second\)%0A%09%7D%0A%7D%0A%60%60%60%0A%E6%AD%A3%E7%A1%AE%E5%A7%BF%E5%8A%BF%E6%98%AF%E5%88%9B%E5%BB%BAtimer%E5%AE%9A%E6%97%B6%E5%99%A8%EF%BC%8C%E6%AF%8F%E6%AC%A1%E9%9C%80%E8%A6%81%E5%90%AF%E5%8A%A8%E5%AE%9A%E6%97%B6%E5%99%A8%E7%9A%84%E6%97%B6%E5%80%99%EF%BC%8C%E4%BD%BF%E7%94%A8Reset%E6%96%B9%E6%B3%95%E9%87%8D%E7%BD%AE%E5%AE%9A%E6%97%B6%E5%99%A8%EF%BC%8C%E8%BF%99%E6%A0%B7%E5%B0%B1%E4%B8%8D%E7%94%A8%E6%AF%8F%E6%AC%A1%E9%83%BD%E8%A6%81%E5%88%9B%E5%BB%BA%E6%96%B0%E7%9A%84%E5%AE%9A%E6%97%B6%E5%99%A8%E4%BA%86%E3%80%82%0A%60%60%60%0Afunc%20goroutineLeak3\(\)%20%7B%0A%09ch%20%3A%3D%20make\(chan%20string\)%0A%09go%20func\(\)%20%7B%0A%09%09timer%20%3A%3D%20time.NewTimer\(3%20\*%20time.Second\)%0A%09%09defer%20timer.Stop\(\)%0A%09%09for%20%7B%0A%09%09%09timer.Reset\(3%20\*%20time.Second\)%0A%09%09%09select%20%7B%0A%09%09%09case%20%3C\-timer.C%3A%0A%09%09%09%09fmt.Println\(%22request%20timeout%22\)%0A%09%09%09case%20data%20%3A%3D%20%3C\-ch%3A%0A%09%09%09%09fmt.Println\(%22%E6%94%B6%E5%88%B0%E6%95%B0%E6%8D%AE%3A%20%22%2C%20data\)%0A%09%09%09%7D%0A%09%09%7D%0A%09%7D\(\)%0A%0A%09for%20i%20%3A%3D%200%3B%20i%20%3C%2010000%3B%20i%2B%2B%20%7B%0A%09%09ch%20%3C\-%20%22ok%22%0A%09%09time.Sleep\(time.Second\)%0A%09%7D%0A%7D%0A%60%60%60%0A%0A%0A%0A%0A%0A%0A%0A
