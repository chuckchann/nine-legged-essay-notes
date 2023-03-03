# Golang之channel源码分析

## 1.  channel

channel是支撑Go语言高性能并发编程模型的重要结构，channel是一个用于同步和通信的**有锁环形队列**，使用互斥锁解决程序中可能存在的线程竞争问题。以下内容基于：

```shell
go version go1.16.2 darwin/amd64
```

## 2. channel的数据结构

channel其是用runtime.hchan来表示的。

```go
type hchan struct {
	qcount   uint           // 缓冲区buffer里有几个元素
	dataqsiz uint           // 缓冲区buffer最多有几个元素 即缓冲区大小
	buf      unsafe.Pointer // 指向底层循环数组的指针
	elemsize uint16         // 元素大小
	closed   uint32        // 是否关闭
	elemtype *_type        // 元素的类型
	sendx    uint         // 已发送元素在环形数组中的索引
	recvx    uint         // 已接收元素在环形数组中的索引
	recvq    waitq        // 等待接收的goroutine队列
	sendq    waitq        // 待发送的goroutine队列
    
	lock mutex            //锁 保护数据
}
```

recvq与sendq是waitq类型的数据，分别代表接受的g队列与发送的g队列，waitq是一个双端链表。

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

buf是一个指向一个环形数组的指针，sendx与recvx分别代表在这个环形数组中的发送位置与接收数据，之所以用环形数组是因为这样可以重复利用空间。

![34ae3951184096a940a3375b1a6ecac1.png](image/34ae3951184096a940a3375b1a6ecac1.png)

整体示意图

![005edda969ff08409fff155b3f84ff8c.png](image/005edda969ff08409fff155b3f84ff8c.png)

---

## 3. 创建一个channel

创建channel最终使用的是runtime.makechan\(\)这个函数。

```go
func makechan(t *chantype, size int) *hchan {
  ...
    
	var c *hchan
	switch {
	case mem == 0: // =>创建无缓冲通道的channel
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0: // =>创建有缓冲通道但元素里不含指针的channel
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        //给buf分配一块连续的内存空间
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default: // =>创建有缓冲通道并且元素含有指针的channel
		c = new(hchan)
        //给buf单独分配内存空间
		c.buf = mallocgc(mem, elem, true)
	}
    ...
}
```

---

## 4. 向channel发送数据

向channel发送数据最终调用的是runtime.chansend，其函数签名为：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    ...
}
```

其中，bolock参数表示遇到channel操作不能成功时是否需要阻塞。往channel发送数据操作不成功的情景：

1. 往无buffer的channel发送数据而接收者还没ready
2. 往buffer已满的channel里发送数据
3. 往nil的channel里发送数据

block为false即不阻塞的情况，如select里的channel操作：

```go
func main() {
    ch := make(chan int)
    select {
        case: ch <- 0:
        default:
    }
    ...
}
```

block为true即阻塞的情况，如往无buffer的channel发送数据：

```go
func main() {
    ch := make(chan int)
    ch <- 0
    ...
}

```

具体看看往一个channel里发送数据的流程（只展示核心流程）。

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

  //省略若干代码...
        
  //非阻塞 且 channel未关闭 且 channel已满
	if !block && c.closed == 0 && full(c) {
		return false
	}

  //操作前先上锁
	lock(&c.lock)
        
  //往已关闭的channel里发送数据会panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  //如果能从接收者的G队列的队头取出一个G，说明已经有接收者ready，也说明了：
  //1.带缓冲的channle的buffer为空，因为如果buffer不为空的话接收者可以直接从buffer里取到数据，不会再将接收者放在接收者队列里
  //2.channel不带缓冲
	if sg := c.recvq.dequeue(); sg != nil {
    //针对上面1.2两种情况 都可以将数据传递给这个接收者 并且唤醒等待者G 
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
    
  //qcount < dataqsiz 表示这是一个带缓冲的channel（如果是无缓冲的话这两个字段都为0）
  //缓冲channel的buffer未满
	if c.qcount < c.dataqsiz {
    //将数据放到缓冲buffer中即可返回
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++ //更新发送索引位置
		if c.sendx == c.dataqsiz {
			c.sendx = 0 //因为是环形的所以重新回到head
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

  //走到这里说明往通道里发送的数据没有把法被对端接收，即：
  //1.往无缓冲的channel发送数据而接收者还没ready
  //2.往有缓冲的channel发送数据但buffer已满
        
  //获取当前G
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg) //将当前的G入队到发送等待队列sendq
	atomic.Store8(&gp.parkingOnChan, 1)
        
  //gopar将当前的G挂起 达到阻塞当前G的效果
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	KeepAlive(ep)

	//数据被接收者接收 当前发送者G被唤醒 后面做一系列收尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
    //如果不是因为接受者ready而是因为channel被close释放当前的G，会导致panic
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

---

## 5. 从channel接收数据

从channel接收数据最终调用的是runtime.chanrecv，其函数签名如下：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool){
    ...
}
```

同样地，block参数表示在channel操作无法完成时是否需要阻塞。从channel接收数据操作不成功的情景：

1. 从无缓冲的channel里接收数据而发送者还没ready
2. 从有缓冲但buffer为空的channel里接受数据
3. 从nil的channel里读取数据

block为false即不阻塞的情况，如select里的channel操作：

```go
func main() {
    ch := make(chan int)
    select {
        case: <-ch:
        default:
    }
    ...
}
```

block为true即阻塞的情况，如从无buffer的channel接收数据：

```go
func main() {
    ch := make(chan int)
    data := <-ch 
    ...
}

```

具体看看从一个channel里接收数据的流程

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	// 省略若干代码... 

  // 从一个nil channel 里接收数据
	if c == nil {
		if !block {
			return //非阻塞  直接返回
		}
    
    //阻塞 当前G会永久阻塞
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
        
  // 省略若干代码... 

  //加锁
	lock(&c.lock)

  // 如果通道被关闭 且 buffer里没有元素 这里直接返回了（返回类型零值）
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil { 
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

  //如果能从发送者G队列取出一个G，说明当前有发送者ready，也说明了：
  //1.带缓冲的channle的buffer已满，因为如果buffer未满的话接收者可以直接往buffer里发送数据，不会再将发送者放到发送者队列里，也就不会有ready的发送者。
  //2.channel不带缓冲
	if sg := c.sendq.dequeue(); sg != nil {
    //1.如果是带缓冲的channel 且buffer已满，除了需要从buffer里去读数据拷贝给接收变量ep外，还需要将发送者G的数据放入buffer，调整发送位置索引， 并且唤醒这个发送者G，以保证FIFO  
    //2.如果是没有缓冲的channel的情况就直接将数据拷贝给接收变量ep
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

  //如果是带缓冲的channel 且buffer里任然有数据了 那么直接从buffer里拿数据即可
	if c.qcount > 0 {
    //qp 即从hcha.buf里获取channel buffer里面的数据
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
    //将qp赋值接收者变量ep
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++ //更新接收索引位置
		if c.recvx == c.dataqsiz {
			c.recvx = 0 //接收队列是环形数组 需重新置位为0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
    
  //到这说明接收者当前没办法接收数据了 即
  //1.不带缓冲channel & 没有发送者发送数据
  //2.不带缓冲channel & buffer里没有数据
    
  //不阻塞的话就返回了
	if !block {
		unlock(&c.lock)
		return false, false
	}
    
    
	// 获取当前G 封装成sodog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg) //将当前G放入接收者的等待队列里

	atomic.Store(&gp.parkingOnChan, 1)
    
  //使用gopark阻塞当前G
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	//有发送者发送数据了 当前G被唤醒
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}

```

------

## 6. 关闭channel

关闭channel最终调用的是runtime.closechan，其函数签名为：

```go
func closechan(c *hchan) {
  //关闭一个nil的channel会panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

  //加锁
	lock(&c.lock)
  
  //关闭一个已关闭的channel会panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}


	//将closed标志位设为1
	c.closed = 1

	var glist gList

	//收集所有等待者队列 将其放到glist队列里并释放
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	//收集所有发送者队列 将其放到glist队列里并释放
  //这里有个坑点：发送者等待队列里面的G唤醒后会导致这个G产生panic，详情可以看前面chansend函数的流程
  //所以close一个chnnel之前要确保所有发送的的数据都被接收者接收了
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	//释放所有的等待者
  //经过前面的分析可知，glist里要么全是发送者等待队列，要么全是接收者等待队列
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

------

## 7. 如何优雅地关闭channel

关于 channel 的使用，有几点不方便的地方：

1. 在不改变 channel 自身状态的情况下，无法获知一个 channel 是否关闭
2. 关闭一个 closed channel 会导致 panic。所以，如果关闭 channel 的一方在不知道 channel 是否处于关闭状态时就去贸然关闭 channel 是很危险的事情。
3. 向一个 closed channel 发送数据会导致 panic。所以，如果向 channel 发送数据的一方不知道 channel 是否处于关闭状态时就去贸然向 channel 发送数据是很危险的事情。

有一条广泛流传的关闭 channel 的原则：

> don’t close a channel from the receiver side and don’t close a channel if the channel has multiple concurrent senders.

不要从一个 receiver 侧关闭 channel，也不要在有多个 sender 时，关闭 channel。比较好理解，向 channel 发送元素的就是 sender，因此 sender 可以决定何时不发送数据，并且关闭 channel。但是如果有多个 sender，某个 sender 同样没法确定其他 sender 的情况，这时也不能贸然关闭 channel。

有两个不那么优雅地关闭 channel 的方法：

1. 使用 defer-recover 机制，放心大胆地关闭 channel 或者向 channel 发送数据。即使发生了 panic，有 defer-recover 在兜底。
2. 使用 sync.Once 来保证只关闭一次。

那么到底如何优雅关闭channel ？根据 sender 和 receiver 的个数，分下面几种情况：

1. 一个 sender，一个 receiver
2. 一个 sender， M 个 receiver
3. N 个 sender，一个 reciver
4. N 个 sender， M 个 receiver

对于1、2 种情况，直接从 sender 端关闭即可，重点要关注的都是第3、4种情况。

### 7.1 *N 个 sender，一个 reciver*

在这种情况下，优雅关闭 channel 的方法是：the only receiver says “please stop sending more” by closing an additional signal channel。即增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 指令。senders 监听到关闭信号后，停止发送数据。

```go
func main()  {
	rand.Seed(time.Now().Unix())

	const NumSenders = 1000
	const max = 1000

	dataChan := make(chan int, 100)
	stopChan := make(chan struct{})

	//senders
	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				select {
				case  <-stopChan:
					return
				case dataChan <- rand.Int():
				}
			}
		}()
	}

	//receiver
	go func() {
		for val := range dataChan {
			if val == max - 1 {
				close(stopChan)
				return
			}
		}
	}()

	select {
	
	}
}

```

这里的 stopCh 就是信号 channel，它本身只有一个 sender，因此可以直接关闭它。senders 收到了关闭信号后，select 分支 “case <- stopCh” 被选中，退出函数，不再发送数据。需要说明的是，上面的代码并没有明确关闭 dataCh。在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。所以，在这种情形下，所谓的优雅地关闭 channel 就是不关闭 channel，让 gc 代劳。

### 7.2 *N 个 sender，M个 reciver*

在这种情况下，优雅关闭 channel 的方法是：any one of them says “let’s end the game” by notifying a moderator to close an additional signal channel。

Todo...







## 8. channel的应用

Channel 和 goroutine 的结合是 Go 并发编程的大杀器。而 channel 的实际应用也经常让人眼前一亮，通过与 select，cancel，timer 等结合，它能实现各种各样的功能。接下来，我们就要梳理一下 channel 的应用。

#### 8.1 *停止信号*

经常是关闭某个 channel 或者向 channel 发送一个元素，使得接收 channel 的那一方获知道此信息，进而做一些其他的操作。

#### 8.2 *任务定时*

如配合timer，设置任务的timeout

```go
select {
	case <-time.After(100 * time.Millisecond):
	case <-s.stopc:
		return false
}
```

如执行某个定时任务

```go
func task() {
	ticker := time.Tick(1 * time.Second)
	for {
		select {
		case <- ticker:
			// 执行定时任务
			fmt.Println("执行 1s 定时任务")
		}
	}
}
```

#### 8.3 *解耦生产方和消费方*

服务启动时，启动 n 个 worker，作为工作协程池，这些协程工作在一个 `for {}` 无限循环里，从某个 channel 消费工作任务并执行：

```go
func main() {
	taskCh := make(chan int, 100)
	go worker(taskCh)

    // 塞任务
	for i := 0; i < 10; i++ {
		taskCh <- i
	}

    // 等待 1 小时 
	select {
	case <-time.After(time.Hour):
	}
}

func worker(taskCh <-chan int) {
	const N = 5
	// 启动 5 个工作协程
	for i := 0; i < N; i++ {
		go func(id int) {
			for {
				task := <- taskCh
				fmt.Printf("finish task: %d by worker %d\n", task, id)
				time.Sleep(time.Second)
			}
		}(i)
	}
}
```

#### 8.4 *控制并发数*

有时需要定时执行几百个任务，例如每天定时按城市来执行一些离线计算的任务。但是并发数又不能太高，因为任务执行过程依赖第三方的一些资源，对请求的速率有限制。这时就可以通过 channel 来控制并发数。

```golang
var limit = make(chan int, 3)

func main() {
    // …………
    for _, w := range work {
        go func() {
            limit <- 1 //试图占坑
            w() //占到坑位 执行w
            <-limit //执行完成 归还坑位
        }()
    }
    // …………
}
```

构建一个缓冲型的 channel，容量为 3。接着遍历任务列表，每个任务启动一个 goroutine 去完成。真正执行任务，访问第三方的动作在 w() 中完成，在执行 w() 之前，先要从 limit 中拿“许可证”，拿到许可证之后，才能执行 w()，并且在执行完任务，要将“许可证”归还。这样就可以控制同时运行的 goroutine 数。

## 9. nil channel 与 closed channel

关于对 nil channel 与 closed channel 的发送、接收、关闭操作，面试经常会考：

|      |      nil channel      | cloesed channel  |
| :--: | :-------------------: | :--------------: |
| 发送 | 永久阻塞(block为ture) |      panic       |
| 接收 | 永久阻塞(block为ture) | 一直返回类型零值 |
| 关闭 |         painc         |      painc       |

