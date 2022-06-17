# 九股文笔记-Golang之context

#### Context是什么

Context是Golang中独特的数据结构，可以用来设置截至日期、同步信号、传递请求相关值，**总地来说就是在goroutine构成的树形结构中对信号进行同步以减少计算资源的浪费**。以下内容基于：

```
go version go1.16.2 darwin/amd64
```

#### Context的数据结构

Context是一个接口，来看下它的接口定义

```
type Context interface {

	//Deadline方法是获取设置的截至时间，到了这个截至时间，
	//Context会自动发起取消，第二个返回值ok代表是否有设置
	//截至时间
 	Deadline() (deadline time.Time, ok bool) 

 	//Done方法返回一个只读的chan，当这个chan收到消息时代
 	//表Parent Context已经发起了取消请求，当前goroutine
 	//要开始做清理操作了
	Done() <-chan struct{} 

	//Err返回Context被取消的原因
	Err() error

	//Value方法返回Context绑定的对应的key的值，这个是线程
	//安全的
	Value(key interface{}) interface{}
}
```

context包提供了6种创建Context的方法。每个方法都会返回一种类型的结构体，这些结构体都实现了Context接口。

![.png](image/.png)

---

#### Context的实现原理

##### emptyCtx

emptyContext是可以从下面看到它其实是一个空实现，即不提供任何功能。

```
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

context.Background\(\) 与 context.TODO\(\) 返回的数据结构都是一个emptyCtx类型的常量，在多数情况下，如果当前函数没有上下文作为入参，我们都会使用 context.Backgroundc\(\)或者context.TODO\(\)作为起始的上下文向下传递。

##### valueCtx

context.WithValue\(\)返回valueCtx的context。valueCtx主要用户上下文中的值传递。

```
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() { //key必须是可以比较的
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context      //父ctx
	key, val interface{}   //当前ctx的key&val
}

func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key) //当前没有ctx没有知道 继续从父ctx找
}
```

这里valueCtx只实现了Context接口里的Value\(\)方法，并没有显式地实现其他三个跟取消相关的方法（因为valueCtx本身也就只有值传递的功能），而是将这些实现交给父ctx，这里算是一种小tricks。

##### cancelCtx

cancelCtx 是整个context实现中**最重要的部分**。重点分析下cancelCtx的实现及一些通用的方法。

```
type cancelCtx struct {
	Context                        //父节点

	mu       sync.Mutex            // 锁 修改下面的三个字段需要上锁
	done     chan struct{}         // 动态创建的channel，将被第一个cancenl()关闭
	children map[canceler]struct{} // children是一个map存储当前节点下的能够被cancel子节点
	err      error                 // err用于存储错误信息 表示任务结束
}

```

cancelCtx本身不是用来传递值的，按理来说cancelCtx的Value方法的应该是直接依赖父ctx来实现，但cancelCtx却显式地实现了Value方法，其原因是需要先判断key是否为cancelCtxKey。这里这个cancelCtxKey是用来判断一个ctx是否为cancelCtx。

```
func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key)
}
```

cancelCtx.Done以懒加载获取一个用于通知的channel，实现如下。

```
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()  //加锁
	if c.done == nil { 
		c.done = make(chan struct{}) //如果c.done为nil就创建 这个done是以懒加载的方式创建的
	}
	d := c.done 
	c.mu.Unlock() //释放锁
	return d
}
```

cancelCtx.Err返回取消的原因。

```
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
```

**Done与Err方法获取结构体里的字段都是有锁机制保护的，所以context可以说是线程安全的**。上面讲了canceCtx的几种方法，下面来说说如何创建cancelCtx。通过context.WihtCancel方法可以创建一个带cancel方法的cancelCtx。context.WithCancel先初始化一个cancelCtx，**再使用context.propagateCancel来告知父ctx，自己的子ctx里出现了一个cancelCtx，需要在他们自己的children字段里加上当前的cancelCtx，然后返回当前cancelCtx及一个cancel函数，cancel函数可以让使用者随时取消当前的cancelCtx**。

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

先来看cancel函数cancelCtx.cancel，其作用是用来取消当前ctx及其子孙ctx。

```
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock() //🔒加锁
	if c.err != nil {
		c.mu.Unlock()
		return     //如果c.err ！= nil 说明这个context已经被其他goroutine取消了 那么直接返回 也验证了多次调用cancel()是安全的
	}
	c.err = err   //设置c.err
	if c.done == nil {  
		c.done = closedchan  //c.done == nil 说明cancel()的时候还没有调用过ctx.Done() 这时将c.done设置为closedchan 意味着这个ctx已经cancel
	} else {
	    //通知所有的引用这个ctx的任务 这里也是利用了关闭通道来批量通知的机制
		close(c.done)     
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err) //cancel掉子ctx
	}
	c.children = nil   //设置childre为nil
	c.mu.Unlock()      //解锁

	if removeFromParent {
		removeChild(c.Context, c) //将当前ctx节点从父节点上移除 
	}
}
```

最后一步是context.removeChild，目的是将当前ctx节点从父节点上移除。

```
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent) //找到父辈中的cancelCtx
	if !ok {       //如果他的父辈中没有cancelCtx 那么后续的操作也没有必要了
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)  //如果他的父辈中有cancelCtx 那么将当前这个child从cancelCtx的children列表中去除
	}
	p.mu.Unlock()
}
```

context.parentCancelCtx 这个方法比较重要,很多地方都有用到，功能是找到父辈中非自定义的cancelCtx\(即只找从context包里出来的\)。

```
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
	if done == closedchan || done == nil {  //说明父ctx已经被取消 不用找了
		return nil, false
	}
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx) //通过cancelCtxKey来判断是否为cancelCtx
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	ok = p.done == done //如果是自定义ctx 也会返回false
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```

context.propagateCancel也是一个重要的方法，其作用是告知父辈ctx，自己的子ctx里出现了一个cancelCtx，需要在他们自己的children字段里加上当前的cancelCtx，以形成父子关系。

```
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {      //父辈ctx不会触发cancel 直接返回 例如父ctx是context.BackGround()
		return // parent is never canceled
	}

	select {
	case <-done:
		child.cancel(false, parent.Err())  //当父ctx已经被canceled(继承一个已经cancel的ctx) 那么当前ctx(即child)挂在父ctx下已经没有意义 直接cancel掉child 
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {  //找到他的父辈中找到cancelCtx
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)      //父ctx已经被canceled
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}  //将当前ctx放到父ctx的children列表里去
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		//走到这个分支说明父辈是非内部实现的ctx(自定义的ctx但实现了Context接口)，并在自定义的Done()返回了非空的管道
		//此时这个自定义的父ctx没法跟当前ctx关联起来，所以需要另外起一个监测的goroutine，去监测父ctx的取消信号
		go func() {
			select {
			case <-parent.Done():     //如果自定义的父ctx发出取消信号 当前ctx也要被cancel
				child.cancel(false, parent.Err())
			case <-child.Done():      //如果当前ctx被cancel 那么也不再需要监测父ctx的取消信号了 这个goroutine就结束了
			}
		}()
	}
}
```

##### timerCtx

先看看timerCtx的结构定义及一些特有的方法。

```
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

可以看出，timerCtx中内嵌了一个cancelCtx，**可见timerCtx取消类相关的功能是依赖cancelCtx来实现的**。context.WithTimeout及context.WithDeadline返回一个timerCtx及取消函数。context.WithTimeout也是基于context.WithDeadline来实现的。

```
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		//父ctx的取消时间在当前ctx之前 说明父ctx会在他的deadline的时候把当前ctx取消掉 所以当前ctx的deadline没什么用了 也不再需要起一个定时器去定时cancel他了
		return WithCancel(parent)  //直接返回一个cancelCtx
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded)               //当前时间已经过了deadline 
		return c, func() { c.cancel(false, Canceled) } //虽然时间已经过了，但还是有可能被父ctx取消掉 所以不用从父ctx的children里摘除
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() { 
			c.cancel(true, DeadlineExceeded)  //启动一个定时器 在deadline的时候cancel	
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

timerCtx的取消函数，当取消函数被调用的时候，把定时器取消掉。

```
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop() //手动取消 停止计时器
		c.timer = nil
	}
	c.mu.Unlock()
```

#### 总结

各种类型的ctx并没有显式地实现接口的各个方法，而是只实现与自己相关的方法。比如，valueCtx只实现了Value方法（这也是它的核心功能），其他的与它本身功能无关的取消类的方法比如（Deadline方法）由它的父ctx实现，这样的设计使得每种类型的ctx只需要关注自己的功能，而无需实现与它功能无关的方法。假设valueCtx也显式地实现Done/Deadline，那么需要递归或者循坏去获取到它的父辈ctx中的cancelCtx或者timerCtx，代码可能如下所示：

```
func (c *valueCtx) Done() <-chan struct{} {
	for cur := c; c != nil; c = c.Context {
		if cancelc, ok := cur.Value(&cancelCtxKey).(*cancelCtx); ok {
			return cancelc.done
		}
	}
	return nil
}
```

这无疑增加了代码的冗余度及让代码的可读性变差。而context包中这种隐式实现的方法让代码变得边界分明，冗余度低，也符合开放原则，在日常的编码中我们可以借鉴。

%23%23%23%23%20Context%E6%98%AF%E4%BB%80%E4%B9%88%0AContext%E6%98%AFGolang%E4%B8%AD%E7%8B%AC%E7%89%B9%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%EF%BC%8C%E5%8F%AF%E4%BB%A5%E7%94%A8%E6%9D%A5%E8%AE%BE%E7%BD%AE%E6%88%AA%E8%87%B3%E6%97%A5%E6%9C%9F%E3%80%81%E5%90%8C%E6%AD%A5%E4%BF%A1%E5%8F%B7%E3%80%81%E4%BC%A0%E9%80%92%E8%AF%B7%E6%B1%82%E7%9B%B8%E5%85%B3%E5%80%BC%EF%BC%8C\*\*%E6%80%BB%E5%9C%B0%E6%9D%A5%E8%AF%B4%E5%B0%B1%E6%98%AF%E5%9C%A8goroutine%E6%9E%84%E6%88%90%E7%9A%84%E6%A0%91%E5%BD%A2%E7%BB%93%E6%9E%84%E4%B8%AD%E5%AF%B9%E4%BF%A1%E5%8F%B7%E8%BF%9B%E8%A1%8C%E5%90%8C%E6%AD%A5%E4%BB%A5%E5%87%8F%E5%B0%91%E8%AE%A1%E7%AE%97%E8%B5%84%E6%BA%90%E7%9A%84%E6%B5%AA%E8%B4%B9\*\*%E3%80%82%E4%BB%A5%E4%B8%8B%E5%86%85%E5%AE%B9%E5%9F%BA%E4%BA%8E%EF%BC%9A%0A%0A%60%60%60%0Ago%20version%20go1.16.2%20darwin%2Famd64%0A%60%60%60%0A%0A%0A%23%23%23%23%20Context%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%0AContext%E6%98%AF%E4%B8%80%E4%B8%AA%E6%8E%A5%E5%8F%A3%EF%BC%8C%E6%9D%A5%E7%9C%8B%E4%B8%8B%E5%AE%83%E7%9A%84%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89%0A%60%60%60%0Atype%20Context%20interface%20%7B%0A%0A%09%2F%2FDeadline%E6%96%B9%E6%B3%95%E6%98%AF%E8%8E%B7%E5%8F%96%E8%AE%BE%E7%BD%AE%E7%9A%84%E6%88%AA%E8%87%B3%E6%97%B6%E9%97%B4%EF%BC%8C%E5%88%B0%E4%BA%86%E8%BF%99%E4%B8%AA%E6%88%AA%E8%87%B3%E6%97%B6%E9%97%B4%EF%BC%8C%0A%09%2F%2FContext%E4%BC%9A%E8%87%AA%E5%8A%A8%E5%8F%91%E8%B5%B7%E5%8F%96%E6%B6%88%EF%BC%8C%E7%AC%AC%E4%BA%8C%E4%B8%AA%E8%BF%94%E5%9B%9E%E5%80%BCok%E4%BB%A3%E8%A1%A8%E6%98%AF%E5%90%A6%E6%9C%89%E8%AE%BE%E7%BD%AE%0A%09%2F%2F%E6%88%AA%E8%87%B3%E6%97%B6%E9%97%B4%0A%20%09Deadline\(\)%20\(deadline%20time.Time%2C%20ok%20bool\)%20%0A%0A%0A%20%09%2F%2FDone%E6%96%B9%E6%B3%95%E8%BF%94%E5%9B%9E%E4%B8%80%E4%B8%AA%E5%8F%AA%E8%AF%BB%E7%9A%84chan%EF%BC%8C%E5%BD%93%E8%BF%99%E4%B8%AAchan%E6%94%B6%E5%88%B0%E6%B6%88%E6%81%AF%E6%97%B6%E4%BB%A3%0A%20%09%2F%2F%E8%A1%A8Parent%20Context%E5%B7%B2%E7%BB%8F%E5%8F%91%E8%B5%B7%E4%BA%86%E5%8F%96%E6%B6%88%E8%AF%B7%E6%B1%82%EF%BC%8C%E5%BD%93%E5%89%8Dgoroutine%0A%20%09%2F%2F%E8%A6%81%E5%BC%80%E5%A7%8B%E5%81%9A%E6%B8%85%E7%90%86%E6%93%8D%E4%BD%9C%E4%BA%86%0A%09Done\(\)%20%3C\-chan%20struct%7B%7D%20%0A%0A%09%2F%2FErr%E8%BF%94%E5%9B%9EContext%E8%A2%AB%E5%8F%96%E6%B6%88%E7%9A%84%E5%8E%9F%E5%9B%A0%0A%09Err\(\)%20error%0A%0A%09%2F%2FValue%E6%96%B9%E6%B3%95%E8%BF%94%E5%9B%9EContext%E7%BB%91%E5%AE%9A%E7%9A%84%E5%AF%B9%E5%BA%94%E7%9A%84key%E7%9A%84%E5%80%BC%EF%BC%8C%E8%BF%99%E4%B8%AA%E6%98%AF%E7%BA%BF%E7%A8%8B%0A%09%2F%2F%E5%AE%89%E5%85%A8%E7%9A%84%0A%09Value\(key%20interface%7B%7D\)%20interface%7B%7D%0A%7D%0A%60%60%60%0Acontext%E5%8C%85%E6%8F%90%E4%BE%9B%E4%BA%866%E7%A7%8D%E5%88%9B%E5%BB%BAContext%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%82%E6%AF%8F%E4%B8%AA%E6%96%B9%E6%B3%95%E9%83%BD%E4%BC%9A%E8%BF%94%E5%9B%9E%E4%B8%80%E7%A7%8D%E7%B1%BB%E5%9E%8B%E7%9A%84%E7%BB%93%E6%9E%84%E4%BD%93%EF%BC%8C%E8%BF%99%E4%BA%9B%E7%BB%93%E6%9E%84%E4%BD%93%E9%83%BD%E5%AE%9E%E7%8E%B0%E4%BA%86Context%E6%8E%A5%E5%8F%A3%E3%80%82%0A\!%5B40402dfb769cc0eeb746e114e3d4a7ee.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp126\)%0A%0A\*%20\*%20\*%0A%23%23%23%23%20Context%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%0A%23%23%23%23%23%20emptyCtx%0AemptyContext%E6%98%AF%E5%8F%AF%E4%BB%A5%E4%BB%8E%E4%B8%8B%E9%9D%A2%E7%9C%8B%E5%88%B0%E5%AE%83%E5%85%B6%E5%AE%9E%E6%98%AF%E4%B8%80%E4%B8%AA%E7%A9%BA%E5%AE%9E%E7%8E%B0%EF%BC%8C%E5%8D%B3%E4%B8%8D%E6%8F%90%E4%BE%9B%E4%BB%BB%E4%BD%95%E5%8A%9F%E8%83%BD%E3%80%82%0A%60%60%60%0Atype%20emptyCtx%20int%0A%0Afunc%20\(\*emptyCtx\)%20Deadline\(\)%20\(deadline%20time.Time%2C%20ok%20bool\)%20%7B%0A%09return%0A%7D%0A%0Afunc%20\(\*emptyCtx\)%20Done\(\)%20%3C\-chan%20struct%7B%7D%20%7B%0A%09return%20nil%0A%7D%0A%0Afunc%20\(\*emptyCtx\)%20Err\(\)%20error%20%7B%0A%09return%20nil%0A%7D%0A%0Afunc%20\(\*emptyCtx\)%20Value\(key%20interface%7B%7D\)%20interface%7B%7D%20%7B%0A%09return%20nil%0A%7D%0A%60%60%60%0A%0Acontext.Background\(\)%20%E4%B8%8E%20context.TODO\(\)%20%E8%BF%94%E5%9B%9E%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E9%83%BD%E6%98%AF%E4%B8%80%E4%B8%AAemptyCtx%E7%B1%BB%E5%9E%8B%E7%9A%84%E5%B8%B8%E9%87%8F%EF%BC%8C%E5%9C%A8%E5%A4%9A%E6%95%B0%E6%83%85%E5%86%B5%E4%B8%8B%EF%BC%8C%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8D%E5%87%BD%E6%95%B0%E6%B2%A1%E6%9C%89%E4%B8%8A%E4%B8%8B%E6%96%87%E4%BD%9C%E4%B8%BA%E5%85%A5%E5%8F%82%EF%BC%8C%E6%88%91%E4%BB%AC%E9%83%BD%E4%BC%9A%E4%BD%BF%E7%94%A8%20context.Backgroundc\(\)%E6%88%96%E8%80%85context.TODO\(\)%E4%BD%9C%E4%B8%BA%E8%B5%B7%E5%A7%8B%E7%9A%84%E4%B8%8A%E4%B8%8B%E6%96%87%E5%90%91%E4%B8%8B%E4%BC%A0%E9%80%92%E3%80%82%0A%0A%23%23%23%23%23%20valueCtx%0Acontext.WithValue\(\)%E8%BF%94%E5%9B%9EvalueCtx%E7%9A%84context%E3%80%82valueCtx%E4%B8%BB%E8%A6%81%E7%94%A8%E6%88%B7%E4%B8%8A%E4%B8%8B%E6%96%87%E4%B8%AD%E7%9A%84%E5%80%BC%E4%BC%A0%E9%80%92%E3%80%82%0A%60%60%60%0Afunc%20WithValue\(parent%20Context%2C%20key%2C%20val%20interface%7B%7D\)%20Context%20%7B%0A%09if%20parent%20%3D%3D%20nil%20%7B%0A%09%09panic\(%22cannot%20create%20context%20from%20nil%20parent%22\)%0A%09%7D%0A%09if%20key%20%3D%3D%20nil%20%7B%0A%09%09panic\(%22nil%20key%22\)%0A%09%7D%0A%09if%20\!reflectlite.TypeOf\(key\).Comparable\(\)%20%7B%20%2F%2Fkey%E5%BF%85%E9%A1%BB%E6%98%AF%E5%8F%AF%E4%BB%A5%E6%AF%94%E8%BE%83%E7%9A%84%0A%09%09panic\(%22key%20is%20not%20comparable%22\)%0A%09%7D%0A%09return%20%26valueCtx%7Bparent%2C%20key%2C%20val%7D%0A%7D%0A%0A%0Atype%20valueCtx%20struct%20%7B%0A%09Context%20%20%20%20%20%20%2F%2F%E7%88%B6ctx%0A%09key%2C%20val%20interface%7B%7D%20%20%20%2F%2F%E5%BD%93%E5%89%8Dctx%E7%9A%84key%26val%0A%7D%0A%0A%0Afunc%20\(c%20\*valueCtx\)%20Value\(key%20interface%7B%7D\)%20interface%7B%7D%20%7B%0A%09if%20c.key%20%3D%3D%20key%20%7B%0A%09%09return%20c.val%0A%09%7D%0A%09return%20c.Context.Value\(key\)%20%2F%2F%E5%BD%93%E5%89%8D%E6%B2%A1%E6%9C%89ctx%E6%B2%A1%E6%9C%89%E7%9F%A5%E9%81%93%20%E7%BB%A7%E7%BB%AD%E4%BB%8E%E7%88%B6ctx%E6%89%BE%0A%7D%0A%60%60%60%0A%E8%BF%99%E9%87%8CvalueCtx%E5%8F%AA%E5%AE%9E%E7%8E%B0%E4%BA%86Context%E6%8E%A5%E5%8F%A3%E9%87%8C%E7%9A%84Value\(\)%E6%96%B9%E6%B3%95%EF%BC%8C%E5%B9%B6%E6%B2%A1%E6%9C%89%E6%98%BE%E5%BC%8F%E5%9C%B0%E5%AE%9E%E7%8E%B0%E5%85%B6%E4%BB%96%E4%B8%89%E4%B8%AA%E8%B7%9F%E5%8F%96%E6%B6%88%E7%9B%B8%E5%85%B3%E7%9A%84%E6%96%B9%E6%B3%95%EF%BC%88%E5%9B%A0%E4%B8%BAvalueCtx%E6%9C%AC%E8%BA%AB%E4%B9%9F%E5%B0%B1%E5%8F%AA%E6%9C%89%E5%80%BC%E4%BC%A0%E9%80%92%E7%9A%84%E5%8A%9F%E8%83%BD%EF%BC%89%EF%BC%8C%E8%80%8C%E6%98%AF%E5%B0%86%E8%BF%99%E4%BA%9B%E5%AE%9E%E7%8E%B0%E4%BA%A4%E7%BB%99%E7%88%B6ctx%EF%BC%8C%E8%BF%99%E9%87%8C%E7%AE%97%E6%98%AF%E4%B8%80%E7%A7%8D%E5%B0%8Ftricks%E3%80%82%0A%0A%23%23%23%23%23%20cancelCtx%0AcancelCtx%20%E6%98%AF%E6%95%B4%E4%B8%AAcontext%E5%AE%9E%E7%8E%B0%E4%B8%AD\*\*%E6%9C%80%E9%87%8D%E8%A6%81%E7%9A%84%E9%83%A8%E5%88%86\*\*%E3%80%82%E9%87%8D%E7%82%B9%E5%88%86%E6%9E%90%E4%B8%8BcancelCtx%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8F%8A%E4%B8%80%E4%BA%9B%E9%80%9A%E7%94%A8%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%82%0A%60%60%60%0Atype%20cancelCtx%20struct%20%7B%0A%09Context%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%E7%88%B6%E8%8A%82%E7%82%B9%0A%0A%09mu%20%20%20%20%20%20%20sync.Mutex%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20%E9%94%81%20%E4%BF%AE%E6%94%B9%E4%B8%8B%E9%9D%A2%E7%9A%84%E4%B8%89%E4%B8%AA%E5%AD%97%E6%AE%B5%E9%9C%80%E8%A6%81%E4%B8%8A%E9%94%81%0A%09done%20%20%20%20%20chan%20struct%7B%7D%20%20%20%20%20%20%20%20%20%2F%2F%20%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%E7%9A%84channel%EF%BC%8C%E5%B0%86%E8%A2%AB%E7%AC%AC%E4%B8%80%E4%B8%AAcancenl\(\)%E5%85%B3%E9%97%AD%0A%09children%20map%5Bcanceler%5Dstruct%7B%7D%20%2F%2F%20children%E6%98%AF%E4%B8%80%E4%B8%AAmap%E5%AD%98%E5%82%A8%E5%BD%93%E5%89%8D%E8%8A%82%E7%82%B9%E4%B8%8B%E7%9A%84%E8%83%BD%E5%A4%9F%E8%A2%ABcancel%E5%AD%90%E8%8A%82%E7%82%B9%0A%09err%20%20%20%20%20%20error%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20err%E7%94%A8%E4%BA%8E%E5%AD%98%E5%82%A8%E9%94%99%E8%AF%AF%E4%BF%A1%E6%81%AF%20%E8%A1%A8%E7%A4%BA%E4%BB%BB%E5%8A%A1%E7%BB%93%E6%9D%9F%0A%7D%0A%0A%60%60%60%0A%0AcancelCtx%E6%9C%AC%E8%BA%AB%E4%B8%8D%E6%98%AF%E7%94%A8%E6%9D%A5%E4%BC%A0%E9%80%92%E5%80%BC%E7%9A%84%EF%BC%8C%E6%8C%89%E7%90%86%E6%9D%A5%E8%AF%B4cancelCtx%E7%9A%84Value%E6%96%B9%E6%B3%95%E7%9A%84%E5%BA%94%E8%AF%A5%E6%98%AF%E7%9B%B4%E6%8E%A5%E4%BE%9D%E8%B5%96%E7%88%B6ctx%E6%9D%A5%E5%AE%9E%E7%8E%B0%EF%BC%8C%E4%BD%86cancelCtx%E5%8D%B4%E6%98%BE%E5%BC%8F%E5%9C%B0%E5%AE%9E%E7%8E%B0%E4%BA%86Value%E6%96%B9%E6%B3%95%EF%BC%8C%E5%85%B6%E5%8E%9F%E5%9B%A0%E6%98%AF%E9%9C%80%E8%A6%81%E5%85%88%E5%88%A4%E6%96%ADkey%E6%98%AF%E5%90%A6%E4%B8%BAcancelCtxKey%E3%80%82%E8%BF%99%E9%87%8C%E8%BF%99%E4%B8%AAcancelCtxKey%E6%98%AF%E7%94%A8%E6%9D%A5%E5%88%A4%E6%96%AD%E4%B8%80%E4%B8%AActx%E6%98%AF%E5%90%A6%E4%B8%BAcancelCtx%E3%80%82%0A%60%60%60%0Afunc%20\(c%20\*cancelCtx\)%20Value\(key%20interface%7B%7D\)%20interface%7B%7D%20%7B%0A%09if%20key%20%3D%3D%20%26cancelCtxKey%20%7B%0A%09%09return%20c%0A%09%7D%0A%09return%20c.Context.Value\(key\)%0A%7D%0A%60%60%60%0A%0AcancelCtx.Done%E4%BB%A5%E6%87%92%E5%8A%A0%E8%BD%BD%E8%8E%B7%E5%8F%96%E4%B8%80%E4%B8%AA%E7%94%A8%E4%BA%8E%E9%80%9A%E7%9F%A5%E7%9A%84channel%EF%BC%8C%E5%AE%9E%E7%8E%B0%E5%A6%82%E4%B8%8B%E3%80%82%0A%60%60%60%0Afunc%20\(c%20\*cancelCtx\)%20Done\(\)%20%3C\-chan%20struct%7B%7D%20%7B%0A%09c.mu.Lock\(\)%20%20%2F%2F%E5%8A%A0%E9%94%81%0A%09if%20c.done%20%3D%3D%20nil%20%7B%20%0A%09%09c.done%20%3D%20make\(chan%20struct%7B%7D\)%20%2F%2F%E5%A6%82%E6%9E%9Cc.done%E4%B8%BAnil%E5%B0%B1%E5%88%9B%E5%BB%BA%20%E8%BF%99%E4%B8%AAdone%E6%98%AF%E4%BB%A5%E6%87%92%E5%8A%A0%E8%BD%BD%E7%9A%84%E6%96%B9%E5%BC%8F%E5%88%9B%E5%BB%BA%E7%9A%84%0A%09%7D%0A%09d%20%3A%3D%20c.done%20%0A%09c.mu.Unlock\(\)%20%2F%2F%E9%87%8A%E6%94%BE%E9%94%81%0A%09return%20d%0A%7D%0A%60%60%60%0A%0AcancelCtx.Err%E8%BF%94%E5%9B%9E%E5%8F%96%E6%B6%88%E7%9A%84%E5%8E%9F%E5%9B%A0%E3%80%82%0A%60%60%60%0Afunc%20\(c%20\*cancelCtx\)%20Err\(\)%20error%20%7B%0A%20%20%20%20c.mu.Lock\(\)%0A%20%20%20%20err%20%3A%3D%20c.err%0A%20%20%20%20c.mu.Unlock\(\)%0A%20%20%20%20return%20err%0A%7D%0A%60%60%60%0A\*\*Done%E4%B8%8EErr%E6%96%B9%E6%B3%95%E8%8E%B7%E5%8F%96%E7%BB%93%E6%9E%84%E4%BD%93%E9%87%8C%E7%9A%84%E5%AD%97%E6%AE%B5%E9%83%BD%E6%98%AF%E6%9C%89%E9%94%81%E6%9C%BA%E5%88%B6%E4%BF%9D%E6%8A%A4%E7%9A%84%EF%BC%8C%E6%89%80%E4%BB%A5context%E5%8F%AF%E4%BB%A5%E8%AF%B4%E6%98%AF%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84\*\*%E3%80%82%E4%B8%8A%E9%9D%A2%E8%AE%B2%E4%BA%86canceCtx%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E6%B3%95%EF%BC%8C%E4%B8%8B%E9%9D%A2%E6%9D%A5%E8%AF%B4%E8%AF%B4%E5%A6%82%E4%BD%95%E5%88%9B%E5%BB%BAcancelCtx%E3%80%82%E9%80%9A%E8%BF%87context.WihtCancel%E6%96%B9%E6%B3%95%E5%8F%AF%E4%BB%A5%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E5%B8%A6cancel%E6%96%B9%E6%B3%95%E7%9A%84cancelCtx%E3%80%82context.WithCancel%E5%85%88%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B8%80%E4%B8%AAcancelCtx%EF%BC%8C\*\*%E5%86%8D%E4%BD%BF%E7%94%A8context.propagateCancel%E6%9D%A5%E5%91%8A%E7%9F%A5%E7%88%B6ctx%EF%BC%8C%E8%87%AA%E5%B7%B1%E7%9A%84%E5%AD%90ctx%E9%87%8C%E5%87%BA%E7%8E%B0%E4%BA%86%E4%B8%80%E4%B8%AAcancelCtx%EF%BC%8C%E9%9C%80%E8%A6%81%E5%9C%A8%E4%BB%96%E4%BB%AC%E8%87%AA%E5%B7%B1%E7%9A%84children%E5%AD%97%E6%AE%B5%E9%87%8C%E5%8A%A0%E4%B8%8A%E5%BD%93%E5%89%8D%E7%9A%84cancelCtx%EF%BC%8C%E7%84%B6%E5%90%8E%E8%BF%94%E5%9B%9E%E5%BD%93%E5%89%8DcancelCtx%E5%8F%8A%E4%B8%80%E4%B8%AAcancel%E5%87%BD%E6%95%B0%EF%BC%8Ccancel%E5%87%BD%E6%95%B0%E5%8F%AF%E4%BB%A5%E8%AE%A9%E4%BD%BF%E7%94%A8%E8%80%85%E9%9A%8F%E6%97%B6%E5%8F%96%E6%B6%88%E5%BD%93%E5%89%8D%E7%9A%84cancelCtx\*\*%E3%80%82%0A%60%60%60%0Afunc%20WithCancel\(parent%20Context\)%20\(ctx%20Context%2C%20cancel%20CancelFunc\)%20%7B%0A%09if%20parent%20%3D%3D%20nil%20%7B%0A%09%09panic\(%22cannot%20create%20context%20from%20nil%20parent%22\)%0A%09%7D%0A%09c%20%3A%3D%20newCancelCtx\(parent\)%0A%09propagateCancel\(parent%2C%20%26c\)%0A%09return%20%26c%2C%20func\(\)%20%7B%20c.cancel\(true%2C%20Canceled\)%20%7D%0A%7D%0A%60%60%60%0A%0A%E5%85%88%E6%9D%A5%E7%9C%8Bcancel%E5%87%BD%E6%95%B0cancelCtx.cancel%EF%BC%8C%E5%85%B6%E4%BD%9C%E7%94%A8%E6%98%AF%E7%94%A8%E6%9D%A5%E5%8F%96%E6%B6%88%E5%BD%93%E5%89%8Dctx%E5%8F%8A%E5%85%B6%E5%AD%90%E5%AD%99ctx%E3%80%82%0A%60%60%60%0Afunc%20\(c%20\*cancelCtx\)%20cancel\(removeFromParent%20bool%2C%20err%20error\)%20%7B%0A%09if%20err%20%3D%3D%20nil%20%7B%0A%09%09panic\(%22context%3A%20internal%20error%3A%20missing%20cancel%20error%22\)%0A%09%7D%0A%09c.mu.Lock\(\)%20%2F%2F%F0%9F%94%92%E5%8A%A0%E9%94%81%0A%09if%20c.err%20\!%3D%20nil%20%7B%0A%09%09c.mu.Unlock\(\)%0A%09%09return%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9Cc.err%20%EF%BC%81%3D%20nil%20%E8%AF%B4%E6%98%8E%E8%BF%99%E4%B8%AAcontext%E5%B7%B2%E7%BB%8F%E8%A2%AB%E5%85%B6%E4%BB%96goroutine%E5%8F%96%E6%B6%88%E4%BA%86%20%E9%82%A3%E4%B9%88%E7%9B%B4%E6%8E%A5%E8%BF%94%E5%9B%9E%20%E4%B9%9F%E9%AA%8C%E8%AF%81%E4%BA%86%E5%A4%9A%E6%AC%A1%E8%B0%83%E7%94%A8cancel\(\)%E6%98%AF%E5%AE%89%E5%85%A8%E7%9A%84%0A%09%7D%0A%09c.err%20%3D%20err%20%20%20%2F%2F%E8%AE%BE%E7%BD%AEc.err%0A%09if%20c.done%20%3D%3D%20nil%20%7B%20%20%0A%09%09c.done%20%3D%20closedchan%20%20%2F%2Fc.done%20%3D%3D%20nil%20%E8%AF%B4%E6%98%8Ecancel\(\)%E7%9A%84%E6%97%B6%E5%80%99%E8%BF%98%E6%B2%A1%E6%9C%89%E8%B0%83%E7%94%A8%E8%BF%87ctx.Done\(\)%20%E8%BF%99%E6%97%B6%E5%B0%86c.done%E8%AE%BE%E7%BD%AE%E4%B8%BAclosedchan%20%E6%84%8F%E5%91%B3%E7%9D%80%E8%BF%99%E4%B8%AActx%E5%B7%B2%E7%BB%8Fcancel%0A%09%7D%20else%20%7B%0A%09%20%20%20%20%2F%2F%E9%80%9A%E7%9F%A5%E6%89%80%E6%9C%89%E7%9A%84%E5%BC%95%E7%94%A8%E8%BF%99%E4%B8%AActx%E7%9A%84%E4%BB%BB%E5%8A%A1%20%E8%BF%99%E9%87%8C%E4%B9%9F%E6%98%AF%E5%88%A9%E7%94%A8%E4%BA%86%E5%85%B3%E9%97%AD%E9%80%9A%E9%81%93%E6%9D%A5%E6%89%B9%E9%87%8F%E9%80%9A%E7%9F%A5%E7%9A%84%E6%9C%BA%E5%88%B6%0A%09%09close\(c.done\)%20%20%20%20%20%0A%09%7D%0A%09for%20child%20%3A%3D%20range%20c.children%20%7B%0A%09%09%2F%2F%20NOTE%3A%20acquiring%20the%20child's%20lock%20while%20holding%20parent's%20lock.%0A%09%09child.cancel\(false%2C%20err\)%20%2F%2Fcancel%E6%8E%89%E5%AD%90ctx%0A%09%7D%0A%09c.children%20%3D%20nil%20%20%20%2F%2F%E8%AE%BE%E7%BD%AEchildre%E4%B8%BAnil%0A%09c.mu.Unlock\(\)%20%20%20%20%20%20%2F%2F%E8%A7%A3%E9%94%81%0A%0A%09if%20removeFromParent%20%7B%0A%09%09removeChild\(c.Context%2C%20c\)%20%2F%2F%E5%B0%86%E5%BD%93%E5%89%8Dctx%E8%8A%82%E7%82%B9%E4%BB%8E%E7%88%B6%E8%8A%82%E7%82%B9%E4%B8%8A%E7%A7%BB%E9%99%A4%20%0A%09%7D%0A%7D%0A%60%60%60%0A%E6%9C%80%E5%90%8E%E4%B8%80%E6%AD%A5%E6%98%AFcontext.removeChild%EF%BC%8C%E7%9B%AE%E7%9A%84%E6%98%AF%E5%B0%86%E5%BD%93%E5%89%8Dctx%E8%8A%82%E7%82%B9%E4%BB%8E%E7%88%B6%E8%8A%82%E7%82%B9%E4%B8%8A%E7%A7%BB%E9%99%A4%E3%80%82%0A%60%60%60%0Afunc%20removeChild\(parent%20Context%2C%20child%20canceler\)%20%7B%0A%09p%2C%20ok%20%3A%3D%20parentCancelCtx\(parent\)%20%2F%2F%E6%89%BE%E5%88%B0%E7%88%B6%E8%BE%88%E4%B8%AD%E7%9A%84cancelCtx%0A%09if%20\!ok%20%7B%20%20%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%E4%BB%96%E7%9A%84%E7%88%B6%E8%BE%88%E4%B8%AD%E6%B2%A1%E6%9C%89cancelCtx%20%E9%82%A3%E4%B9%88%E5%90%8E%E7%BB%AD%E7%9A%84%E6%93%8D%E4%BD%9C%E4%B9%9F%E6%B2%A1%E6%9C%89%E5%BF%85%E8%A6%81%E4%BA%86%0A%09%09return%0A%09%7D%0A%09p.mu.Lock\(\)%0A%09if%20p.children%20\!%3D%20nil%20%7B%0A%09%09delete\(p.children%2C%20child\)%20%20%2F%2F%E5%A6%82%E6%9E%9C%E4%BB%96%E7%9A%84%E7%88%B6%E8%BE%88%E4%B8%AD%E6%9C%89cancelCtx%20%E9%82%A3%E4%B9%88%E5%B0%86%E5%BD%93%E5%89%8D%E8%BF%99%E4%B8%AAchild%E4%BB%8EcancelCtx%E7%9A%84children%E5%88%97%E8%A1%A8%E4%B8%AD%E5%8E%BB%E9%99%A4%0A%09%7D%0A%09p.mu.Unlock\(\)%0A%7D%0A%60%60%60%0A%20context.parentCancelCtx%20%E8%BF%99%E4%B8%AA%E6%96%B9%E6%B3%95%E6%AF%94%E8%BE%83%E9%87%8D%E8%A6%81%2C%E5%BE%88%E5%A4%9A%E5%9C%B0%E6%96%B9%E9%83%BD%E6%9C%89%E7%94%A8%E5%88%B0%EF%BC%8C%E5%8A%9F%E8%83%BD%E6%98%AF%E6%89%BE%E5%88%B0%E7%88%B6%E8%BE%88%E4%B8%AD%E9%9D%9E%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84cancelCtx\(%E5%8D%B3%E5%8F%AA%E6%89%BE%E4%BB%8Econtext%E5%8C%85%E9%87%8C%E5%87%BA%E6%9D%A5%E7%9A%84\)%E3%80%82%0A%20%0A%60%60%60%0Afunc%20parentCancelCtx\(parent%20Context\)%20\(\*cancelCtx%2C%20bool\)%20%7B%0A%09done%20%3A%3D%20parent.Done\(\)%0A%09if%20done%20%3D%3D%20closedchan%20%7C%7C%20done%20%3D%3D%20nil%20%7B%20%20%2F%2F%E8%AF%B4%E6%98%8E%E7%88%B6ctx%E5%B7%B2%E7%BB%8F%E8%A2%AB%E5%8F%96%E6%B6%88%20%E4%B8%8D%E7%94%A8%E6%89%BE%E4%BA%86%0A%09%09return%20nil%2C%20false%0A%09%7D%0A%09p%2C%20ok%20%3A%3D%20parent.Value\(%26cancelCtxKey\).\(\*cancelCtx\)%20%2F%2F%E9%80%9A%E8%BF%87cancelCtxKey%E6%9D%A5%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E4%B8%BAcancelCtx%0A%09if%20\!ok%20%7B%0A%09%09return%20nil%2C%20false%0A%09%7D%0A%09p.mu.Lock\(\)%0A%09ok%20%3D%20p.done%20%3D%3D%20done%20%2F%2F%E5%A6%82%E6%9E%9C%E6%98%AF%E8%87%AA%E5%AE%9A%E4%B9%89ctx%20%E4%B9%9F%E4%BC%9A%E8%BF%94%E5%9B%9Efalse%0A%09p.mu.Unlock\(\)%0A%09if%20\!ok%20%7B%0A%09%09return%20nil%2C%20false%0A%09%7D%0A%09return%20p%2C%20true%0A%7D%0A%60%60%60%0Acontext.propagateCancel%E4%B9%9F%E6%98%AF%E4%B8%80%E4%B8%AA%E9%87%8D%E8%A6%81%E7%9A%84%E6%96%B9%E6%B3%95%EF%BC%8C%E5%85%B6%E4%BD%9C%E7%94%A8%E6%98%AF%E5%91%8A%E7%9F%A5%E7%88%B6%E8%BE%88ctx%EF%BC%8C%E8%87%AA%E5%B7%B1%E7%9A%84%E5%AD%90ctx%E9%87%8C%E5%87%BA%E7%8E%B0%E4%BA%86%E4%B8%80%E4%B8%AAcancelCtx%EF%BC%8C%E9%9C%80%E8%A6%81%E5%9C%A8%E4%BB%96%E4%BB%AC%E8%87%AA%E5%B7%B1%E7%9A%84children%E5%AD%97%E6%AE%B5%E9%87%8C%E5%8A%A0%E4%B8%8A%E5%BD%93%E5%89%8D%E7%9A%84cancelCtx%EF%BC%8C%E4%BB%A5%E5%BD%A2%E6%88%90%E7%88%B6%E5%AD%90%E5%85%B3%E7%B3%BB%E3%80%82%0A%60%60%60%0Afunc%20propagateCancel\(parent%20Context%2C%20child%20canceler\)%20%7B%0A%09done%20%3A%3D%20parent.Done\(\)%0A%09if%20done%20%3D%3D%20nil%20%7B%20%20%20%20%20%20%2F%2F%E7%88%B6%E8%BE%88ctx%E4%B8%8D%E4%BC%9A%E8%A7%A6%E5%8F%91cancel%20%E7%9B%B4%E6%8E%A5%E8%BF%94%E5%9B%9E%20%E4%BE%8B%E5%A6%82%E7%88%B6ctx%E6%98%AFcontext.BackGround\(\)%0A%09%09return%20%2F%2F%20parent%20is%20never%20canceled%0A%09%7D%0A%0A%09select%20%7B%0A%09case%20%3C\-done%3A%0A%09%09child.cancel\(false%2C%20parent.Err\(\)\)%20%20%2F%2F%E5%BD%93%E7%88%B6ctx%E5%B7%B2%E7%BB%8F%E8%A2%ABcanceled\(%E7%BB%A7%E6%89%BF%E4%B8%80%E4%B8%AA%E5%B7%B2%E7%BB%8Fcancel%E7%9A%84ctx\)%20%E9%82%A3%E4%B9%88%E5%BD%93%E5%89%8Dctx\(%E5%8D%B3child\)%E6%8C%82%E5%9C%A8%E7%88%B6ctx%E4%B8%8B%E5%B7%B2%E7%BB%8F%E6%B2%A1%E6%9C%89%E6%84%8F%E4%B9%89%20%E7%9B%B4%E6%8E%A5cancel%E6%8E%89child%20%0A%09%09return%0A%09default%3A%0A%09%7D%0A%0A%09if%20p%2C%20ok%20%3A%3D%20parentCancelCtx\(parent\)%3B%20ok%20%7B%20%20%2F%2F%E6%89%BE%E5%88%B0%E4%BB%96%E7%9A%84%E7%88%B6%E8%BE%88%E4%B8%AD%E6%89%BE%E5%88%B0cancelCtx%0A%09%09p.mu.Lock\(\)%0A%09%09if%20p.err%20\!%3D%20nil%20%7B%0A%09%09%09%2F%2F%20parent%20has%20already%20been%20canceled%0A%09%09%09child.cancel\(false%2C%20p.err\)%20%20%20%20%20%20%2F%2F%E7%88%B6ctx%E5%B7%B2%E7%BB%8F%E8%A2%ABcanceled%0A%09%09%7D%20else%20%7B%0A%09%09%09if%20p.children%20%3D%3D%20nil%20%7B%0A%09%09%09%09p.children%20%3D%20make\(map%5Bcanceler%5Dstruct%7B%7D\)%0A%09%09%09%7D%0A%09%09%09p.children%5Bchild%5D%20%3D%20struct%7B%7D%7B%7D%20%20%2F%2F%E5%B0%86%E5%BD%93%E5%89%8Dctx%E6%94%BE%E5%88%B0%E7%88%B6ctx%E7%9A%84children%E5%88%97%E8%A1%A8%E9%87%8C%E5%8E%BB%0A%09%09%7D%0A%09%09p.mu.Unlock\(\)%0A%09%7D%20else%20%7B%0A%09%09atomic.AddInt32\(%26goroutines%2C%20%2B1\)%0A%09%09%2F%2F%E8%B5%B0%E5%88%B0%E8%BF%99%E4%B8%AA%E5%88%86%E6%94%AF%E8%AF%B4%E6%98%8E%E7%88%B6%E8%BE%88%E6%98%AF%E9%9D%9E%E5%86%85%E9%83%A8%E5%AE%9E%E7%8E%B0%E7%9A%84ctx\(%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84ctx%E4%BD%86%E5%AE%9E%E7%8E%B0%E4%BA%86Context%E6%8E%A5%E5%8F%A3\)%EF%BC%8C%E5%B9%B6%E5%9C%A8%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84Done\(\)%E8%BF%94%E5%9B%9E%E4%BA%86%E9%9D%9E%E7%A9%BA%E7%9A%84%E7%AE%A1%E9%81%93%0A%09%09%2F%2F%E6%AD%A4%E6%97%B6%E8%BF%99%E4%B8%AA%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84%E7%88%B6ctx%E6%B2%A1%E6%B3%95%E8%B7%9F%E5%BD%93%E5%89%8Dctx%E5%85%B3%E8%81%94%E8%B5%B7%E6%9D%A5%EF%BC%8C%E6%89%80%E4%BB%A5%E9%9C%80%E8%A6%81%E5%8F%A6%E5%A4%96%E8%B5%B7%E4%B8%80%E4%B8%AA%E7%9B%91%E6%B5%8B%E7%9A%84goroutine%EF%BC%8C%E5%8E%BB%E7%9B%91%E6%B5%8B%E7%88%B6ctx%E7%9A%84%E5%8F%96%E6%B6%88%E4%BF%A1%E5%8F%B7%0A%09%09go%20func\(\)%20%7B%0A%09%09%09select%20%7B%0A%09%09%09case%20%3C\-parent.Done\(\)%3A%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84%E7%88%B6ctx%E5%8F%91%E5%87%BA%E5%8F%96%E6%B6%88%E4%BF%A1%E5%8F%B7%20%E5%BD%93%E5%89%8Dctx%E4%B9%9F%E8%A6%81%E8%A2%ABcancel%0A%09%09%09%09child.cancel\(false%2C%20parent.Err\(\)\)%0A%09%09%09case%20%3C\-child.Done\(\)%3A%20%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8Dctx%E8%A2%ABcancel%20%E9%82%A3%E4%B9%88%E4%B9%9F%E4%B8%8D%E5%86%8D%E9%9C%80%E8%A6%81%E7%9B%91%E6%B5%8B%E7%88%B6ctx%E7%9A%84%E5%8F%96%E6%B6%88%E4%BF%A1%E5%8F%B7%E4%BA%86%20%E8%BF%99%E4%B8%AAgoroutine%E5%B0%B1%E7%BB%93%E6%9D%9F%E4%BA%86%0A%09%09%09%7D%0A%09%09%7D\(\)%0A%09%7D%0A%7D%0A%60%60%60%0A%23%23%23%23%23%20timerCtx%0A%E5%85%88%E7%9C%8B%E7%9C%8BtimerCtx%E7%9A%84%E7%BB%93%E6%9E%84%E5%AE%9A%E4%B9%89%E5%8F%8A%E4%B8%80%E4%BA%9B%E7%89%B9%E6%9C%89%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%82%0A%60%60%60%0Atype%20timerCtx%20struct%20%7B%0A%09cancelCtx%0A%09timer%20\*time.Timer%20%2F%2F%20Under%20cancelCtx.mu.%0A%0A%09deadline%20time.Time%0A%7D%0A%60%60%60%0A%E5%8F%AF%E4%BB%A5%E7%9C%8B%E5%87%BA%EF%BC%8CtimerCtx%E4%B8%AD%E5%86%85%E5%B5%8C%E4%BA%86%E4%B8%80%E4%B8%AAcancelCtx%EF%BC%8C\*\*%E5%8F%AF%E8%A7%81timerCtx%E5%8F%96%E6%B6%88%E7%B1%BB%E7%9B%B8%E5%85%B3%E7%9A%84%E5%8A%9F%E8%83%BD%E6%98%AF%E4%BE%9D%E8%B5%96cancelCtx%E6%9D%A5%E5%AE%9E%E7%8E%B0%E7%9A%84\*\*%E3%80%82context.WithTimeout%E5%8F%8Acontext.WithDeadline%E8%BF%94%E5%9B%9E%E4%B8%80%E4%B8%AAtimerCtx%E5%8F%8A%E5%8F%96%E6%B6%88%E5%87%BD%E6%95%B0%E3%80%82context.WithTimeout%E4%B9%9F%E6%98%AF%E5%9F%BA%E4%BA%8Econtext.WithDeadline%E6%9D%A5%E5%AE%9E%E7%8E%B0%E7%9A%84%E3%80%82%0A%60%60%60%0Afunc%20WithTimeout\(parent%20Context%2C%20timeout%20time.Duration\)%20\(Context%2C%20CancelFunc\)%20%7B%0A%09return%20WithDeadline\(parent%2C%20time.Now\(\).Add\(timeout\)\)%0A%7D%0A%0A%0Afunc%20WithDeadline\(parent%20Context%2C%20d%20time.Time\)%20\(Context%2C%20CancelFunc\)%20%7B%0A%09if%20parent%20%3D%3D%20nil%20%7B%0A%09%09panic\(%22cannot%20create%20context%20from%20nil%20parent%22\)%0A%09%7D%0A%09if%20cur%2C%20ok%20%3A%3D%20parent.Deadline\(\)%3B%20ok%20%26%26%20cur.Before\(d\)%20%7B%0A%09%09%2F%2F%E7%88%B6ctx%E7%9A%84%E5%8F%96%E6%B6%88%E6%97%B6%E9%97%B4%E5%9C%A8%E5%BD%93%E5%89%8Dctx%E4%B9%8B%E5%89%8D%20%E8%AF%B4%E6%98%8E%E7%88%B6ctx%E4%BC%9A%E5%9C%A8%E4%BB%96%E7%9A%84deadline%E7%9A%84%E6%97%B6%E5%80%99%E6%8A%8A%E5%BD%93%E5%89%8Dctx%E5%8F%96%E6%B6%88%E6%8E%89%20%E6%89%80%E4%BB%A5%E5%BD%93%E5%89%8Dctx%E7%9A%84deadline%E6%B2%A1%E4%BB%80%E4%B9%88%E7%94%A8%E4%BA%86%20%E4%B9%9F%E4%B8%8D%E5%86%8D%E9%9C%80%E8%A6%81%E8%B5%B7%E4%B8%80%E4%B8%AA%E5%AE%9A%E6%97%B6%E5%99%A8%E5%8E%BB%E5%AE%9A%E6%97%B6cancel%E4%BB%96%E4%BA%86%0A%09%09return%20WithCancel\(parent\)%20%20%2F%2F%E7%9B%B4%E6%8E%A5%E8%BF%94%E5%9B%9E%E4%B8%80%E4%B8%AAcancelCtx%0A%09%7D%0A%09c%20%3A%3D%20%26timerCtx%7B%0A%09%09cancelCtx%3A%20newCancelCtx\(parent\)%2C%0A%09%09deadline%3A%20%20d%2C%0A%09%7D%0A%09propagateCancel\(parent%2C%20c\)%0A%09dur%20%3A%3D%20time.Until\(d\)%0A%09if%20dur%20%3C%3D%200%20%7B%0A%09%09c.cancel\(true%2C%20DeadlineExceeded\)%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%E5%BD%93%E5%89%8D%E6%97%B6%E9%97%B4%E5%B7%B2%E7%BB%8F%E8%BF%87%E4%BA%86deadline%20%0A%09%09return%20c%2C%20func\(\)%20%7B%20c.cancel\(false%2C%20Canceled\)%20%7D%20%2F%2F%E8%99%BD%E7%84%B6%E6%97%B6%E9%97%B4%E5%B7%B2%E7%BB%8F%E8%BF%87%E4%BA%86%EF%BC%8C%E4%BD%86%E8%BF%98%E6%98%AF%E6%9C%89%E5%8F%AF%E8%83%BD%E8%A2%AB%E7%88%B6ctx%E5%8F%96%E6%B6%88%E6%8E%89%20%E6%89%80%E4%BB%A5%E4%B8%8D%E7%94%A8%E4%BB%8E%E7%88%B6ctx%E7%9A%84children%E9%87%8C%E6%91%98%E9%99%A4%0A%09%7D%0A%09c.mu.Lock\(\)%0A%09defer%20c.mu.Unlock\(\)%0A%09if%20c.err%20%3D%3D%20nil%20%7B%0A%09%09c.timer%20%3D%20time.AfterFunc\(dur%2C%20func\(\)%20%7B%20%0A%09%09%09c.cancel\(true%2C%20DeadlineExceeded\)%20%20%2F%2F%E5%90%AF%E5%8A%A8%E4%B8%80%E4%B8%AA%E5%AE%9A%E6%97%B6%E5%99%A8%20%E5%9C%A8deadline%E7%9A%84%E6%97%B6%E5%80%99cancel%09%0A%09%09%7D\)%0A%09%7D%0A%09return%20c%2C%20func\(\)%20%7B%20c.cancel\(true%2C%20Canceled\)%20%7D%0A%7D%0A%60%60%60%0AtimerCtx%E7%9A%84%E5%8F%96%E6%B6%88%E5%87%BD%E6%95%B0%EF%BC%8C%E5%BD%93%E5%8F%96%E6%B6%88%E5%87%BD%E6%95%B0%E8%A2%AB%E8%B0%83%E7%94%A8%E7%9A%84%E6%97%B6%E5%80%99%EF%BC%8C%E6%8A%8A%E5%AE%9A%E6%97%B6%E5%99%A8%E5%8F%96%E6%B6%88%E6%8E%89%E3%80%82%0A%60%60%60%0A%09c.cancelCtx.cancel\(false%2C%20err\)%0A%09if%20removeFromParent%20%7B%0A%09%09%2F%2F%20Remove%20this%20timerCtx%20from%20its%20parent%20cancelCtx's%20children.%0A%09%09removeChild\(c.cancelCtx.Context%2C%20c\)%0A%09%7D%0A%09c.mu.Lock\(\)%0A%09if%20c.timer%20\!%3D%20nil%20%7B%0A%09%09c.timer.Stop\(\)%20%2F%2F%E6%89%8B%E5%8A%A8%E5%8F%96%E6%B6%88%20%E5%81%9C%E6%AD%A2%E8%AE%A1%E6%97%B6%E5%99%A8%0A%09%09c.timer%20%3D%20nil%0A%09%7D%0A%09c.mu.Unlock\(\)%0A%60%60%60%0A%0A%23%23%23%23%20%E6%80%BB%E7%BB%93%0A%E5%90%84%E7%A7%8D%E7%B1%BB%E5%9E%8B%E7%9A%84ctx%E5%B9%B6%E6%B2%A1%E6%9C%89%E6%98%BE%E5%BC%8F%E5%9C%B0%E5%AE%9E%E7%8E%B0%E6%8E%A5%E5%8F%A3%E7%9A%84%E5%90%84%E4%B8%AA%E6%96%B9%E6%B3%95%EF%BC%8C%E8%80%8C%E6%98%AF%E5%8F%AA%E5%AE%9E%E7%8E%B0%E4%B8%8E%E8%87%AA%E5%B7%B1%E7%9B%B8%E5%85%B3%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%82%E6%AF%94%E5%A6%82%EF%BC%8CvalueCtx%E5%8F%AA%E5%AE%9E%E7%8E%B0%E4%BA%86Value%E6%96%B9%E6%B3%95%EF%BC%88%E8%BF%99%E4%B9%9F%E6%98%AF%E5%AE%83%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%EF%BC%89%EF%BC%8C%E5%85%B6%E4%BB%96%E7%9A%84%E4%B8%8E%E5%AE%83%E6%9C%AC%E8%BA%AB%E5%8A%9F%E8%83%BD%E6%97%A0%E5%85%B3%E7%9A%84%E5%8F%96%E6%B6%88%E7%B1%BB%E7%9A%84%E6%96%B9%E6%B3%95%E6%AF%94%E5%A6%82%EF%BC%88Deadline%E6%96%B9%E6%B3%95%EF%BC%89%E7%94%B1%E5%AE%83%E7%9A%84%E7%88%B6ctx%E5%AE%9E%E7%8E%B0%EF%BC%8C%E8%BF%99%E6%A0%B7%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%BD%BF%E5%BE%97%E6%AF%8F%E7%A7%8D%E7%B1%BB%E5%9E%8B%E7%9A%84ctx%E5%8F%AA%E9%9C%80%E8%A6%81%E5%85%B3%E6%B3%A8%E8%87%AA%E5%B7%B1%E7%9A%84%E5%8A%9F%E8%83%BD%EF%BC%8C%E8%80%8C%E6%97%A0%E9%9C%80%E5%AE%9E%E7%8E%B0%E4%B8%8E%E5%AE%83%E5%8A%9F%E8%83%BD%E6%97%A0%E5%85%B3%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%82%E5%81%87%E8%AE%BEvalueCtx%E4%B9%9F%E6%98%BE%E5%BC%8F%E5%9C%B0%E5%AE%9E%E7%8E%B0Done%2FDeadline%EF%BC%8C%E9%82%A3%E4%B9%88%E9%9C%80%E8%A6%81%E9%80%92%E5%BD%92%E6%88%96%E8%80%85%E5%BE%AA%E5%9D%8F%E5%8E%BB%E8%8E%B7%E5%8F%96%E5%88%B0%E5%AE%83%E7%9A%84%E7%88%B6%E8%BE%88ctx%E4%B8%AD%E7%9A%84cancelCtx%E6%88%96%E8%80%85timerCtx%EF%BC%8C%E4%BB%A3%E7%A0%81%E5%8F%AF%E8%83%BD%E5%A6%82%E4%B8%8B%E6%89%80%E7%A4%BA%EF%BC%9A%0A%60%60%60%0Afunc%20\(c%20\*valueCtx\)%20Done\(\)%20%3C\-chan%20struct%7B%7D%20%7B%0A%09for%20cur%20%3A%3D%20c%3B%20c%20\!%3D%20nil%3B%20c%20%3D%20c.Context%20%7B%0A%09%09if%20cancelc%2C%20ok%20%3A%3D%20cur.Value\(%26cancelCtxKey\).\(\*cancelCtx\)%3B%20ok%20%7B%0A%09%09%09return%20cancelc.done%0A%09%09%7D%0A%09%7D%0A%09return%20nil%0A%7D%0A%60%60%60%0A%E8%BF%99%E6%97%A0%E7%96%91%E5%A2%9E%E5%8A%A0%E4%BA%86%E4%BB%A3%E7%A0%81%E7%9A%84%E5%86%97%E4%BD%99%E5%BA%A6%E5%8F%8A%E8%AE%A9%E4%BB%A3%E7%A0%81%E7%9A%84%E5%8F%AF%E8%AF%BB%E6%80%A7%E5%8F%98%E5%B7%AE%E3%80%82%E8%80%8Ccontext%E5%8C%85%E4%B8%AD%E8%BF%99%E7%A7%8D%E9%9A%90%E5%BC%8F%E5%AE%9E%E7%8E%B0%E7%9A%84%E6%96%B9%E6%B3%95%E8%AE%A9%E4%BB%A3%E7%A0%81%E5%8F%98%E5%BE%97%E8%BE%B9%E7%95%8C%E5%88%86%E6%98%8E%EF%BC%8C%E5%86%97%E4%BD%99%E5%BA%A6%E4%BD%8E%EF%BC%8C%E4%B9%9F%E7%AC%A6%E5%90%88%E5%BC%80%E6%94%BE%E5%8E%9F%E5%88%99%EF%BC%8C%E5%9C%A8%E6%97%A5%E5%B8%B8%E7%9A%84%E7%BC%96%E7%A0%81%E4%B8%AD%E6%88%91%E4%BB%AC%E5%8F%AF%E4%BB%A5%E5%80%9F%E9%89%B4%E3%80%82
