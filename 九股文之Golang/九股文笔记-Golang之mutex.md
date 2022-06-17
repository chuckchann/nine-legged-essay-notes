# 九股文笔记-Golang之mutex

#### 互斥锁Mutex

在Golang中用于表示互斥锁的是sync.Lock，其作用是保护临界区，确保任意时间只有一个goroutine能拿到锁。

---

#### 正常模式&饥饿模式

为了保证公平性，Golang在v1.9的互斥锁版本中引入了饥饿模式与正常模式。

1. 如果当前锁正在被持有，抢不到锁就会进入一个等待的队列，当锁被释放后，从这个队列的队头里唤醒一个goroutine（等待者），但是锁不会直接给这个等待者，而是必须与**正在获取锁但还未进入等待队列**的goroutine竞争这把锁，与刚唤醒的等待者相比，这个goroutine正持有CPU，所以获取到锁的概率较大。如果等待者抢锁失败，那么它就会被放回队列头部，如果它超过1ms都还没获取到锁，就从**正常模式**切换为**饥饿模式**。
2. 在饥饿模式下，当锁释放后，锁会直接交给等待队列的第一个等待者，不必再与新来的goroutine竞争，新来的goroutine会直接加到等待队列的队尾。当满足以下两个条件时，**饥饿模式**将切换回**正常模式**。
    （1）当前被唤醒的等待者获得锁后，发现队列只剩它自己一个了，那么**切换回正常模式**。
    （2）当前被唤醒的等待者获得锁后，发现自己的等待时间不超过1ms，那么**切换回正常模式**。

在正常模式下，当前拥有CPU的goroutine比起等待队列里的goroutine有很大几率获得锁，这样可以避免协程上下文的频繁切换。但这样又会导致等待队列里的goroutine活活“饿死”，所以又必须有饥饿模式，保证等待已久的goroutine能够获取到锁。

---

#### sync.Lock

```
type Mutex struct {
	state int32 //表示锁的状态，有锁定、饥饿、唤醒等状态
	sema  uint32 //表示信号量 用于实现mutex阻塞队列的定位
}

const (
	mutexLocked = 1 << iota //1 state & 1 == 1 表示上锁状态
	mutexWoken	            //2 state & 2 == 1 表示唤醒状态 
	mutexStarving           //4 state & 4 == 1 表示饥饿状态
	mutexWaiterShift = iota //3 state >> 3 获取等待者的数量	

	starvationThresholdNs = 1e6 //进入饥饿状态的阈值
)
```

state字段总共占用32个bit，其中用前三位表示三个状态，后29个bit表示等待的goroutine个数。

![d9c038b6df9ba92d2d89491fbb0453f8.png](image/d9c038b6df9ba92d2d89491fbb0453f8.png)

* locked: 表示互斥锁的锁定状态。
* woken: 表示从正常模式被唤醒。
* strving: 表示互斥锁进入饥饿状态。
* waiterCount: 表示互斥锁上等待goroutine的个数。

---

#### Lock\(\)加锁过程

```
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

通过CAS判断锁的当前状态，如果state的第一个位为0，那么说明此锁没有被占领，可以直接获取到锁。如果当前锁已被占领，则进入lockSlow\(\)阶段。lockSlow\(\)可以分为如下几个阶段：

1. 判断当前Goroutine能否自旋
2. 通过自旋转等待互斥锁的释放
3. 计算互斥锁的最新状态
4. 更新互斥锁的状态并获取锁

```
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
    //当前不为饥饿状态并 且 runtime_canSpin()返回true 才能进入自旋状态
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
            //runtime_doSpin()自旋函数，会执行30次PAUSE指令，该指令只会占用CPU并消耗CPU时间
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
```

自旋是一种多线程同步机制，自旋的线程会一直占用CPU，直到达成某个条件。自旋可以防止CPU切换到其他线程上，避免了上下文切换，但线程长期自旋会使长期CPU空转，其他线程的任务没法执行。所以要进入自旋状态需要满足两个条件:

1. state&\(mutexLocked|mutexStarving\)返回true，即锁处于正常模式。
2. runtime\_canSpin\(\)返回true:

* 运行在多 CPU 的机器。
* 当前 Goroutine 为了获取该锁进入自旋的次数小于四次。
* 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空

```
        //如果没有进入自旋转状态
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
        //如果不是饥饿模式，new标记为上锁状态
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
        //如果
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
```

%23%23%23%23%20%E4%BA%92%E6%96%A5%E9%94%81Mutex%0A%E5%9C%A8Golang%E4%B8%AD%E7%94%A8%E4%BA%8E%E8%A1%A8%E7%A4%BA%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E6%98%AFsync.Lock%EF%BC%8C%E5%85%B6%E4%BD%9C%E7%94%A8%E6%98%AF%E4%BF%9D%E6%8A%A4%E4%B8%B4%E7%95%8C%E5%8C%BA%EF%BC%8C%E7%A1%AE%E4%BF%9D%E4%BB%BB%E6%84%8F%E6%97%B6%E9%97%B4%E5%8F%AA%E6%9C%89%E4%B8%80%E4%B8%AAgoroutine%E8%83%BD%E6%8B%BF%E5%88%B0%E9%94%81%E3%80%82%0A%0A%0A\*%20\*%20\*%0A%0A%23%23%23%23%20%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F%26%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F%0A%E4%B8%BA%E4%BA%86%E4%BF%9D%E8%AF%81%E5%85%AC%E5%B9%B3%E6%80%A7%EF%BC%8CGolang%E5%9C%A8v1.9%E7%9A%84%E4%BA%92%E6%96%A5%E9%94%81%E7%89%88%E6%9C%AC%E4%B8%AD%E5%BC%95%E5%85%A5%E4%BA%86%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F%E4%B8%8E%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F%E3%80%82%0A%0A1.%20%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8D%E9%94%81%E6%AD%A3%E5%9C%A8%E8%A2%AB%E6%8C%81%E6%9C%89%EF%BC%8C%E6%8A%A2%E4%B8%8D%E5%88%B0%E9%94%81%E5%B0%B1%E4%BC%9A%E8%BF%9B%E5%85%A5%E4%B8%80%E4%B8%AA%E7%AD%89%E5%BE%85%E7%9A%84%E9%98%9F%E5%88%97%EF%BC%8C%E5%BD%93%E9%94%81%E8%A2%AB%E9%87%8A%E6%94%BE%E5%90%8E%EF%BC%8C%E4%BB%8E%E8%BF%99%E4%B8%AA%E9%98%9F%E5%88%97%E7%9A%84%E9%98%9F%E5%A4%B4%E9%87%8C%E5%94%A4%E9%86%92%E4%B8%80%E4%B8%AAgoroutine%EF%BC%88%E7%AD%89%E5%BE%85%E8%80%85%EF%BC%89%EF%BC%8C%E4%BD%86%E6%98%AF%E9%94%81%E4%B8%8D%E4%BC%9A%E7%9B%B4%E6%8E%A5%E7%BB%99%E8%BF%99%E4%B8%AA%E7%AD%89%E5%BE%85%E8%80%85%EF%BC%8C%E8%80%8C%E6%98%AF%E5%BF%85%E9%A1%BB%E4%B8%8E\*\*%E6%AD%A3%E5%9C%A8%E8%8E%B7%E5%8F%96%E9%94%81%E4%BD%86%E8%BF%98%E6%9C%AA%E8%BF%9B%E5%85%A5%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97\*\*%E7%9A%84goroutine%E7%AB%9E%E4%BA%89%E8%BF%99%E6%8A%8A%E9%94%81%EF%BC%8C%E4%B8%8E%E5%88%9A%E5%94%A4%E9%86%92%E7%9A%84%E7%AD%89%E5%BE%85%E8%80%85%E7%9B%B8%E6%AF%94%EF%BC%8C%E8%BF%99%E4%B8%AAgoroutine%E6%AD%A3%E6%8C%81%E6%9C%89CPU%EF%BC%8C%E6%89%80%E4%BB%A5%E8%8E%B7%E5%8F%96%E5%88%B0%E9%94%81%E7%9A%84%E6%A6%82%E7%8E%87%E8%BE%83%E5%A4%A7%E3%80%82%E5%A6%82%E6%9E%9C%E7%AD%89%E5%BE%85%E8%80%85%E6%8A%A2%E9%94%81%E5%A4%B1%E8%B4%A5%EF%BC%8C%E9%82%A3%E4%B9%88%E5%AE%83%E5%B0%B1%E4%BC%9A%E8%A2%AB%E6%94%BE%E5%9B%9E%E9%98%9F%E5%88%97%E5%A4%B4%E9%83%A8%EF%BC%8C%E5%A6%82%E6%9E%9C%E5%AE%83%E8%B6%85%E8%BF%871ms%E9%83%BD%E8%BF%98%E6%B2%A1%E8%8E%B7%E5%8F%96%E5%88%B0%E9%94%81%EF%BC%8C%E5%B0%B1%E4%BB%8E\*\*%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F\*\*%E5%88%87%E6%8D%A2%E4%B8%BA\*\*%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F\*\*%E3%80%82%0A2.%20%E5%9C%A8%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F%E4%B8%8B%EF%BC%8C%E5%BD%93%E9%94%81%E9%87%8A%E6%94%BE%E5%90%8E%EF%BC%8C%E9%94%81%E4%BC%9A%E7%9B%B4%E6%8E%A5%E4%BA%A4%E7%BB%99%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E7%AD%89%E5%BE%85%E8%80%85%EF%BC%8C%E4%B8%8D%E5%BF%85%E5%86%8D%E4%B8%8E%E6%96%B0%E6%9D%A5%E7%9A%84goroutine%E7%AB%9E%E4%BA%89%EF%BC%8C%E6%96%B0%E6%9D%A5%E7%9A%84goroutine%E4%BC%9A%E7%9B%B4%E6%8E%A5%E5%8A%A0%E5%88%B0%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97%E7%9A%84%E9%98%9F%E5%B0%BE%E3%80%82%E5%BD%93%E6%BB%A1%E8%B6%B3%E4%BB%A5%E4%B8%8B%E4%B8%A4%E4%B8%AA%E6%9D%A1%E4%BB%B6%E6%97%B6%EF%BC%8C\*\*%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F\*\*%E5%B0%86%E5%88%87%E6%8D%A2%E5%9B%9E\*\*%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F\*\*%E3%80%82%0A%EF%BC%881%EF%BC%89%E5%BD%93%E5%89%8D%E8%A2%AB%E5%94%A4%E9%86%92%E7%9A%84%E7%AD%89%E5%BE%85%E8%80%85%E8%8E%B7%E5%BE%97%E9%94%81%E5%90%8E%EF%BC%8C%E5%8F%91%E7%8E%B0%E9%98%9F%E5%88%97%E5%8F%AA%E5%89%A9%E5%AE%83%E8%87%AA%E5%B7%B1%E4%B8%80%E4%B8%AA%E4%BA%86%EF%BC%8C%E9%82%A3%E4%B9%88\*\*%E5%88%87%E6%8D%A2%E5%9B%9E%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F\*\*%E3%80%82%0A%EF%BC%882%EF%BC%89%E5%BD%93%E5%89%8D%E8%A2%AB%E5%94%A4%E9%86%92%E7%9A%84%E7%AD%89%E5%BE%85%E8%80%85%E8%8E%B7%E5%BE%97%E9%94%81%E5%90%8E%EF%BC%8C%E5%8F%91%E7%8E%B0%E8%87%AA%E5%B7%B1%E7%9A%84%E7%AD%89%E5%BE%85%E6%97%B6%E9%97%B4%E4%B8%8D%E8%B6%85%E8%BF%871ms%EF%BC%8C%E9%82%A3%E4%B9%88\*\*%E5%88%87%E6%8D%A2%E5%9B%9E%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F\*\*%E3%80%82%0A%0A%E5%9C%A8%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F%E4%B8%8B%EF%BC%8C%E5%BD%93%E5%89%8D%E6%8B%A5%E6%9C%89CPU%E7%9A%84goroutine%E6%AF%94%E8%B5%B7%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97%E9%87%8C%E7%9A%84goroutine%E6%9C%89%E5%BE%88%E5%A4%A7%E5%87%A0%E7%8E%87%E8%8E%B7%E5%BE%97%E9%94%81%EF%BC%8C%E8%BF%99%E6%A0%B7%E5%8F%AF%E4%BB%A5%E9%81%BF%E5%85%8D%E5%8D%8F%E7%A8%8B%E4%B8%8A%E4%B8%8B%E6%96%87%E7%9A%84%E9%A2%91%E7%B9%81%E5%88%87%E6%8D%A2%E3%80%82%E4%BD%86%E8%BF%99%E6%A0%B7%E5%8F%88%E4%BC%9A%E5%AF%BC%E8%87%B4%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97%E9%87%8C%E7%9A%84goroutine%E6%B4%BB%E6%B4%BB%E2%80%9C%E9%A5%BF%E6%AD%BB%E2%80%9D%EF%BC%8C%E6%89%80%E4%BB%A5%E5%8F%88%E5%BF%85%E9%A1%BB%E6%9C%89%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F%EF%BC%8C%E4%BF%9D%E8%AF%81%E7%AD%89%E5%BE%85%E5%B7%B2%E4%B9%85%E7%9A%84goroutine%E8%83%BD%E5%A4%9F%E8%8E%B7%E5%8F%96%E5%88%B0%E9%94%81%E3%80%82%0A%0A%20%20%20%20%20%20%20%0A%0A\*%20\*%20\*%0A%23%23%23%23%20sync.Lock%0A%0A%60%60%60%0Atype%20Mutex%20struct%20%7B%0A%09state%20int32%20%2F%2F%E8%A1%A8%E7%A4%BA%E9%94%81%E7%9A%84%E7%8A%B6%E6%80%81%EF%BC%8C%E6%9C%89%E9%94%81%E5%AE%9A%E3%80%81%E9%A5%A5%E9%A5%BF%E3%80%81%E5%94%A4%E9%86%92%E7%AD%89%E7%8A%B6%E6%80%81%0A%09sema%20%20uint32%20%2F%2F%E8%A1%A8%E7%A4%BA%E4%BF%A1%E5%8F%B7%E9%87%8F%20%E7%94%A8%E4%BA%8E%E5%AE%9E%E7%8E%B0mutex%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97%E7%9A%84%E5%AE%9A%E4%BD%8D%0A%7D%0A%0Aconst%20\(%0A%09mutexLocked%20%3D%201%20%3C%3C%20iota%20%2F%2F1%20state%20%26%201%20%3D%3D%201%20%E8%A1%A8%E7%A4%BA%E4%B8%8A%E9%94%81%E7%8A%B6%E6%80%81%0A%09mutexWoken%09%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F2%20state%20%26%202%20%3D%3D%201%20%E8%A1%A8%E7%A4%BA%E5%94%A4%E9%86%92%E7%8A%B6%E6%80%81%20%0A%09mutexStarving%20%20%20%20%20%20%20%20%20%20%20%2F%2F4%20state%20%26%204%20%3D%3D%201%20%E8%A1%A8%E7%A4%BA%E9%A5%A5%E9%A5%BF%E7%8A%B6%E6%80%81%0A%09mutexWaiterShift%20%3D%20iota%20%2F%2F3%20state%20%3E%3E%203%20%E8%8E%B7%E5%8F%96%E7%AD%89%E5%BE%85%E8%80%85%E7%9A%84%E6%95%B0%E9%87%8F%09%0A%0A%09starvationThresholdNs%20%3D%201e6%20%2F%2F%E8%BF%9B%E5%85%A5%E9%A5%A5%E9%A5%BF%E7%8A%B6%E6%80%81%E7%9A%84%E9%98%88%E5%80%BC%0A\)%0A%60%60%60%0A%0A%0Astate%E5%AD%97%E6%AE%B5%E6%80%BB%E5%85%B1%E5%8D%A0%E7%94%A832%E4%B8%AAbit%EF%BC%8C%E5%85%B6%E4%B8%AD%E7%94%A8%E5%89%8D%E4%B8%89%E4%BD%8D%E8%A1%A8%E7%A4%BA%E4%B8%89%E4%B8%AA%E7%8A%B6%E6%80%81%EF%BC%8C%E5%90%8E29%E4%B8%AAbit%E8%A1%A8%E7%A4%BA%E7%AD%89%E5%BE%85%E7%9A%84goroutine%E4%B8%AA%E6%95%B0%E3%80%82%0A\!%5Bd9c038b6df9ba92d2d89491fbb0453f8.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp82\)%0A%0A%0A\*%20locked%3A%20%E8%A1%A8%E7%A4%BA%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E9%94%81%E5%AE%9A%E7%8A%B6%E6%80%81%E3%80%82%0A\*%20woken%3A%20%E8%A1%A8%E7%A4%BA%E4%BB%8E%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F%E8%A2%AB%E5%94%A4%E9%86%92%E3%80%82%0A\*%20strving%3A%20%E8%A1%A8%E7%A4%BA%E4%BA%92%E6%96%A5%E9%94%81%E8%BF%9B%E5%85%A5%E9%A5%A5%E9%A5%BF%E7%8A%B6%E6%80%81%E3%80%82%0A\*%20waiterCount%3A%20%E8%A1%A8%E7%A4%BA%E4%BA%92%E6%96%A5%E9%94%81%E4%B8%8A%E7%AD%89%E5%BE%85goroutine%E7%9A%84%E4%B8%AA%E6%95%B0%E3%80%82%0A%0A%0A\*%20\*%20\*%0A%0A%0A%23%23%23%23%20Lock\(\)%E5%8A%A0%E9%94%81%E8%BF%87%E7%A8%8B%0A%60%60%60%0Afunc%20\(m%20\*Mutex\)%20Lock\(\)%20%7B%0A%09%2F%2F%20Fast%20path%3A%20grab%20unlocked%20mutex.%0A%09if%20atomic.CompareAndSwapInt32\(%26m.state%2C%200%2C%20mutexLocked\)%20%7B%0A%09%09if%20race.Enabled%20%7B%0A%09%09%09race.Acquire\(unsafe.Pointer\(m\)\)%0A%09%09%7D%0A%09%09return%0A%09%7D%0A%09%2F%2F%20Slow%20path%20\(outlined%20so%20that%20the%20fast%20path%20can%20be%20inlined\)%0A%09m.lockSlow\(\)%0A%7D%0A%60%60%60%0A%E9%80%9A%E8%BF%87CAS%E5%88%A4%E6%96%AD%E9%94%81%E7%9A%84%E5%BD%93%E5%89%8D%E7%8A%B6%E6%80%81%EF%BC%8C%E5%A6%82%E6%9E%9Cstate%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E4%BD%8D%E4%B8%BA0%EF%BC%8C%E9%82%A3%E4%B9%88%E8%AF%B4%E6%98%8E%E6%AD%A4%E9%94%81%E6%B2%A1%E6%9C%89%E8%A2%AB%E5%8D%A0%E9%A2%86%EF%BC%8C%E5%8F%AF%E4%BB%A5%E7%9B%B4%E6%8E%A5%E8%8E%B7%E5%8F%96%E5%88%B0%E9%94%81%E3%80%82%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8D%E9%94%81%E5%B7%B2%E8%A2%AB%E5%8D%A0%E9%A2%86%EF%BC%8C%E5%88%99%E8%BF%9B%E5%85%A5lockSlow\(\)%E9%98%B6%E6%AE%B5%E3%80%82lockSlow\(\)%E5%8F%AF%E4%BB%A5%E5%88%86%E4%B8%BA%E5%A6%82%E4%B8%8B%E5%87%A0%E4%B8%AA%E9%98%B6%E6%AE%B5%EF%BC%9A%0A%0A1.%20%E5%88%A4%E6%96%AD%E5%BD%93%E5%89%8DGoroutine%E8%83%BD%E5%90%A6%E8%87%AA%E6%97%8B%0A2.%20%E9%80%9A%E8%BF%87%E8%87%AA%E6%97%8B%E8%BD%AC%E7%AD%89%E5%BE%85%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E9%87%8A%E6%94%BE%0A3.%20%E8%AE%A1%E7%AE%97%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E6%9C%80%E6%96%B0%E7%8A%B6%E6%80%81%0A4.%20%E6%9B%B4%E6%96%B0%E4%BA%92%E6%96%A5%E9%94%81%E7%9A%84%E7%8A%B6%E6%80%81%E5%B9%B6%E8%8E%B7%E5%8F%96%E9%94%81%0A%0A%60%60%60%0Afunc%20\(m%20\*Mutex\)%20lockSlow\(\)%20%7B%0A%09var%20waitStartTime%20int64%0A%09starving%20%3A%3D%20false%0A%09awoke%20%3A%3D%20false%0A%09iter%20%3A%3D%200%0A%09old%20%3A%3D%20m.state%0A%09for%20%7B%0A%20%20%20%20%2F%2F%E5%BD%93%E5%89%8D%E4%B8%8D%E4%B8%BA%E9%A5%A5%E9%A5%BF%E7%8A%B6%E6%80%81%E5%B9%B6%20%E4%B8%94%20runtime\_canSpin\(\)%E8%BF%94%E5%9B%9Etrue%20%E6%89%8D%E8%83%BD%E8%BF%9B%E5%85%A5%E8%87%AA%E6%97%8B%E7%8A%B6%E6%80%81%0A%09%09if%20old%26\(mutexLocked%7CmutexStarving\)%20%3D%3D%20mutexLocked%20%26%26%20runtime\_canSpin\(iter\)%20%7B%0A%09%09%09if%20\!awoke%20%26%26%20old%26mutexWoken%20%3D%3D%200%20%26%26%20old%3E%3EmutexWaiterShift%20\!%3D%200%20%26%26%0A%09%09%09%09atomic.CompareAndSwapInt32\(%26m.state%2C%20old%2C%20old%7CmutexWoken\)%20%7B%0A%09%09%09%09awoke%20%3D%20true%0A%09%09%09%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%2F%2Fruntime\_doSpin\(\)%E8%87%AA%E6%97%8B%E5%87%BD%E6%95%B0%EF%BC%8C%E4%BC%9A%E6%89%A7%E8%A1%8C30%E6%AC%A1PAUSE%E6%8C%87%E4%BB%A4%EF%BC%8C%E8%AF%A5%E6%8C%87%E4%BB%A4%E5%8F%AA%E4%BC%9A%E5%8D%A0%E7%94%A8CPU%E5%B9%B6%E6%B6%88%E8%80%97CPU%E6%97%B6%E9%97%B4%0A%09%09%09runtime\_doSpin\(\)%0A%09%09%09iter%2B%2B%0A%09%09%09old%20%3D%20m.state%0A%09%09%09continue%0A%09%09%7D%0A%60%60%60%0A%E8%87%AA%E6%97%8B%E6%98%AF%E4%B8%80%E7%A7%8D%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E6%9C%BA%E5%88%B6%EF%BC%8C%E8%87%AA%E6%97%8B%E7%9A%84%E7%BA%BF%E7%A8%8B%E4%BC%9A%E4%B8%80%E7%9B%B4%E5%8D%A0%E7%94%A8CPU%EF%BC%8C%E7%9B%B4%E5%88%B0%E8%BE%BE%E6%88%90%E6%9F%90%E4%B8%AA%E6%9D%A1%E4%BB%B6%E3%80%82%E8%87%AA%E6%97%8B%E5%8F%AF%E4%BB%A5%E9%98%B2%E6%AD%A2CPU%E5%88%87%E6%8D%A2%E5%88%B0%E5%85%B6%E4%BB%96%E7%BA%BF%E7%A8%8B%E4%B8%8A%EF%BC%8C%E9%81%BF%E5%85%8D%E4%BA%86%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2%EF%BC%8C%E4%BD%86%E7%BA%BF%E7%A8%8B%E9%95%BF%E6%9C%9F%E8%87%AA%E6%97%8B%E4%BC%9A%E4%BD%BF%E9%95%BF%E6%9C%9FCPU%E7%A9%BA%E8%BD%AC%EF%BC%8C%E5%85%B6%E4%BB%96%E7%BA%BF%E7%A8%8B%E7%9A%84%E4%BB%BB%E5%8A%A1%E6%B2%A1%E6%B3%95%E6%89%A7%E8%A1%8C%E3%80%82%E6%89%80%E4%BB%A5%E8%A6%81%E8%BF%9B%E5%85%A5%E8%87%AA%E6%97%8B%E7%8A%B6%E6%80%81%E9%9C%80%E8%A6%81%E6%BB%A1%E8%B6%B3%E4%B8%A4%E4%B8%AA%E6%9D%A1%E4%BB%B6%3A%0A%0A1.%20state%26\(mutexLocked%7CmutexStarving\)%E8%BF%94%E5%9B%9Etrue%EF%BC%8C%E5%8D%B3%E9%94%81%E5%A4%84%E4%BA%8E%E6%AD%A3%E5%B8%B8%E6%A8%A1%E5%BC%8F%E3%80%82%0A2.%20runtime\_canSpin\(\)%E8%BF%94%E5%9B%9Etrue%3A%0A\*%20%20%20%20%E8%BF%90%E8%A1%8C%E5%9C%A8%E5%A4%9A%20CPU%20%E7%9A%84%E6%9C%BA%E5%99%A8%E3%80%82%0A\*%20%20%20%20%E5%BD%93%E5%89%8D%20Goroutine%20%E4%B8%BA%E4%BA%86%E8%8E%B7%E5%8F%96%E8%AF%A5%E9%94%81%E8%BF%9B%E5%85%A5%E8%87%AA%E6%97%8B%E7%9A%84%E6%AC%A1%E6%95%B0%E5%B0%8F%E4%BA%8E%E5%9B%9B%E6%AC%A1%E3%80%82%0A\*%20%20%20%20%E5%BD%93%E5%89%8D%E6%9C%BA%E5%99%A8%E4%B8%8A%E8%87%B3%E5%B0%91%E5%AD%98%E5%9C%A8%E4%B8%80%E4%B8%AA%E6%AD%A3%E5%9C%A8%E8%BF%90%E8%A1%8C%E7%9A%84%E5%A4%84%E7%90%86%E5%99%A8%20P%20%E5%B9%B6%E4%B8%94%E5%A4%84%E7%90%86%E7%9A%84%E8%BF%90%E8%A1%8C%E9%98%9F%E5%88%97%E4%B8%BA%E7%A9%BA%20%20%0A%0A%60%60%60%0A%20%20%20%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%E6%B2%A1%E6%9C%89%E8%BF%9B%E5%85%A5%E8%87%AA%E6%97%8B%E8%BD%AC%E7%8A%B6%E6%80%81%0A%09%09new%20%3A%3D%20old%0A%09%09%2F%2F%20Don't%20try%20to%20acquire%20starving%20mutex%2C%20new%20arriving%20goroutines%20must%20queue.%0A%20%20%20%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%E4%B8%8D%E6%98%AF%E9%A5%A5%E9%A5%BF%E6%A8%A1%E5%BC%8F%EF%BC%8Cnew%E6%A0%87%E8%AE%B0%E4%B8%BA%E4%B8%8A%E9%94%81%E7%8A%B6%E6%80%81%0A%09%09if%20old%26mutexStarving%20%3D%3D%200%20%7B%0A%09%09%09new%20%7C%3D%20mutexLocked%0A%09%09%7D%0A%20%20%20%20%20%20%20%20%2F%2F%E5%A6%82%E6%9E%9C%0A%09%09if%20old%26\(mutexLocked%7CmutexStarving\)%20\!%3D%200%20%7B%0A%09%09%09new%20%2B%3D%201%20%3C%3C%20mutexWaiterShift%0A%09%09%7D%0A%09%09%2F%2F%20The%20current%20goroutine%20switches%20mutex%20to%20starvation%20mode.%0A%09%09%2F%2F%20But%20if%20the%20mutex%20is%20currently%20unlocked%2C%20don't%20do%20the%20switch.%0A%09%09%2F%2F%20Unlock%20expects%20that%20starving%20mutex%20has%20waiters%2C%20which%20will%20not%0A%09%09%2F%2F%20be%20true%20in%20this%20case.%0A%09%09if%20starving%20%26%26%20old%26mutexLocked%20\!%3D%200%20%7B%0A%09%09%09new%20%7C%3D%20mutexStarving%0A%09%09%7D%0A%09%09if%20awoke%20%7B%0A%09%09%09%2F%2F%20The%20goroutine%20has%20been%20woken%20from%20sleep%2C%0A%09%09%09%2F%2F%20so%20we%20need%20to%20reset%20the%20flag%20in%20either%20case.%0A%09%09%09if%20new%26mutexWoken%20%3D%3D%200%20%7B%0A%09%09%09%09throw\(%22sync%3A%20inconsistent%20mutex%20state%22\)%0A%09%09%09%7D%0A%09%09%09new%20%26%5E%3D%20mutexWoken%0A%09%09%7D%0A%60%60%60%0A%0A
