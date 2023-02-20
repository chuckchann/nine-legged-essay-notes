# Golang之sync.Once源码分析

### sync.Once简介

sync.Once 是 Go 语言实现的一种对象，用来**保证某种行为只会被执行一次。**sync.Once通常用来实现单例模式的初始化。

### 源码分析

以下内容基于：

```shell
go version go1.16.2 darwin/amd64
```

sync.Once结构

```go
type Once struct {
	done uint32  //用来表示是否已经执行 0-未执行 1-以执行
	m    Mutex   //互斥锁
}
```

Sync.Once只提供了一个Do的方法。

```go
func (o *Once) Do(f func()) {
  // 一种错误的方式是使用cas锁 atomic.CompareAndSwapUint32
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
  // cas锁虽然保证了原子操作，但可能会出现Do里的流程还没执行完就返回的情况
  // 比如：A号groutine先执行了cas操作，将o.done改为1，然后执行Do流程，B号goroutine
  // 也执行cas操作，发现o.done已经为1了，直接返回，但这时Do流程可能还未执行完！sync.Once
  // 是要保证函数返回后Do流程一定是执行完的，所以这违反了它设计的原则。

	if atomic.LoadUint32(&o.done) == 0 { //如果 o.done==1 则说明方法已执行 直接返回 
		// 如果是0 说明还未执行 进入执行doSlow流程（注意！！！这里可能有好几个goroutine进入这里去执行）
		o.doSlow(f)
	}
}
```

once.doSlow是正真执行Do流程的函数。

```go
func (o *Once) doSlow(f func()) {
  //前面说了在并发的情况下 会有好几个goroutine进入这里去执行 所以要先加锁
	o.m.Lock()
	defer o.m.Unlock()
  //明明前面已经获取了锁 这里为什么还要判断o.done == 0 ?
  //其实这里是类似单例模式里的double-check
  //这里假设有 a b c 三个goroutine进入了doSlow，其中a先获取到锁，然后执行do流程
  //这时b c 阻塞
  //等a执行完，释放锁，然后b拿到锁，此时b已经不需要再执行do流程了，所以这里仍然需要判断o.done == 0
  //c同理 
  //这个流程就是类似单例模式里的double-check的流程
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

