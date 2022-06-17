# 九股文笔记-Golang之sync.Map

#### map的并发操作

从前面文章的分析可以看到，原生的map如果并发读写的话，会抛出异常。

```
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
```

也就是说原生的map不是并发安全的。如果要实现并发安全，可以在操作map的时候加一把mutex锁。但官方提供了更高效的sync.Map，使其能在并发安全的前提下更加高效地读写。

---

#### sync.Map数据结构

```
type Map struct {

	mu Mutex //互斥锁 保护dirty字段

	read atomic.Value //只读数据 实际类型为sync.readOnly

	dirty map[interface{}]*entry //写入数据 操作前需先加锁
 
	misses int //每次从read读取失败（read穿透） misses+1
}
```

其中sync.readOnly的数据结构

```
type readOnly struct {
	m       map[interface{}]*entry
	amended bool //如果dirty里存在m中没有的key 这个值为true
}
```

sync.entry用来存储指向value的指针

```
type entry struct {
	p unsafe.Pointer // *interface{}
}
```

其中，read与dirty里各自维护着一套key，key指向的都是同一个entry，也就是说只要修改了这个entry，那么read与dirty都是可见的。

![f61a2a3233069db6cf8faf3dc54c3a8b.png](image/f61a2a3233069db6cf8faf3dc54c3a8b.png)

entry.p有三种状态，分别是：

1. p == nil
2. p == expunged
3. 其他

---

#### sync.Map原理

* 通过read与dirty两个字段，将读写操作分离，读的数据只存在只读字段read上，新写入的数据则放到dirty中。
* 读取的时候会先读read，read中没有再查询dirty。
* 读取read不加锁，但操作dirty需要mutex保护。
* misses字段表示read读取失败但是在dirty读到（read被穿透）的次数，当misses达到一定值，需要将dirty升级为read，已减少miss的次数。
* 对于删除的数据通过标记来延迟删除。

#### sync.Map操作

%23%23%23%23%20map%E7%9A%84%E5%B9%B6%E5%8F%91%E6%93%8D%E4%BD%9C%0A%0A%E4%BB%8E%E5%89%8D%E9%9D%A2%E6%96%87%E7%AB%A0%E7%9A%84%E5%88%86%E6%9E%90%E5%8F%AF%E4%BB%A5%E7%9C%8B%E5%88%B0%EF%BC%8C%E5%8E%9F%E7%94%9F%E7%9A%84map%E5%A6%82%E6%9E%9C%E5%B9%B6%E5%8F%91%E8%AF%BB%E5%86%99%E7%9A%84%E8%AF%9D%EF%BC%8C%E4%BC%9A%E6%8A%9B%E5%87%BA%E5%BC%82%E5%B8%B8%E3%80%82%0A%60%60%60%0A%09if%20h.flags%26hashWriting%20\!%3D%200%20%7B%0A%09%09throw\(%22concurrent%20map%20read%20and%20map%20write%22\)%0A%09%7D%0A%60%60%60%0A%0A%E4%B9%9F%E5%B0%B1%E6%98%AF%E8%AF%B4%E5%8E%9F%E7%94%9F%E7%9A%84map%E4%B8%8D%E6%98%AF%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E7%9A%84%E3%80%82%E5%A6%82%E6%9E%9C%E8%A6%81%E5%AE%9E%E7%8E%B0%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%9C%A8%E6%93%8D%E4%BD%9Cmap%E7%9A%84%E6%97%B6%E5%80%99%E5%8A%A0%E4%B8%80%E6%8A%8Amutex%E9%94%81%E3%80%82%E4%BD%86%E5%AE%98%E6%96%B9%E6%8F%90%E4%BE%9B%E4%BA%86%E6%9B%B4%E9%AB%98%E6%95%88%E7%9A%84sync.Map%EF%BC%8C%E4%BD%BF%E5%85%B6%E8%83%BD%E5%9C%A8%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E7%9A%84%E5%89%8D%E6%8F%90%E4%B8%8B%E6%9B%B4%E5%8A%A0%E9%AB%98%E6%95%88%E5%9C%B0%E8%AF%BB%E5%86%99%E3%80%82%0A%0A\*%20\*%20\*%0A%23%23%23%23%20sync.Map%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%0A%60%60%60%0Atype%20Map%20struct%20%7B%0A%0A%09mu%20Mutex%20%2F%2F%E4%BA%92%E6%96%A5%E9%94%81%20%E4%BF%9D%E6%8A%A4dirty%E5%AD%97%E6%AE%B5%0A%0A%09read%20atomic.Value%20%2F%2F%E5%8F%AA%E8%AF%BB%E6%95%B0%E6%8D%AE%20%E5%AE%9E%E9%99%85%E7%B1%BB%E5%9E%8B%E4%B8%BAsync.readOnly%0A%0A%09dirty%20map%5Binterface%7B%7D%5D\*entry%20%2F%2F%E5%86%99%E5%85%A5%E6%95%B0%E6%8D%AE%20%E6%93%8D%E4%BD%9C%E5%89%8D%E9%9C%80%E5%85%88%E5%8A%A0%E9%94%81%0A%20%0A%09misses%20int%20%2F%2F%E6%AF%8F%E6%AC%A1%E4%BB%8Eread%E8%AF%BB%E5%8F%96%E5%A4%B1%E8%B4%A5%EF%BC%88read%E7%A9%BF%E9%80%8F%EF%BC%89%20misses%2B1%0A%7D%0A%60%60%60%0A%E5%85%B6%E4%B8%ADsync.readOnly%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%0A%60%60%60%0Atype%20readOnly%20struct%20%7B%0A%09m%20%20%20%20%20%20%20map%5Binterface%7B%7D%5D\*entry%0A%09amended%20bool%20%2F%2F%E5%A6%82%E6%9E%9Cdirty%E9%87%8C%E5%AD%98%E5%9C%A8m%E4%B8%AD%E6%B2%A1%E6%9C%89%E7%9A%84key%20%E8%BF%99%E4%B8%AA%E5%80%BC%E4%B8%BAtrue%0A%7D%0A%60%60%60%0Async.entry%E7%94%A8%E6%9D%A5%E5%AD%98%E5%82%A8%E6%8C%87%E5%90%91value%E7%9A%84%E6%8C%87%E9%92%88%0A%60%60%60%0Atype%20entry%20struct%20%7B%0A%09p%20unsafe.Pointer%20%2F%2F%20\*interface%7B%7D%0A%7D%0A%60%60%60%0A%E5%85%B6%E4%B8%AD%EF%BC%8Cread%E4%B8%8Edirty%E9%87%8C%E5%90%84%E8%87%AA%E7%BB%B4%E6%8A%A4%E7%9D%80%E4%B8%80%E5%A5%97key%EF%BC%8Ckey%E6%8C%87%E5%90%91%E7%9A%84%E9%83%BD%E6%98%AF%E5%90%8C%E4%B8%80%E4%B8%AAentry%EF%BC%8C%E4%B9%9F%E5%B0%B1%E6%98%AF%E8%AF%B4%E5%8F%AA%E8%A6%81%E4%BF%AE%E6%94%B9%E4%BA%86%E8%BF%99%E4%B8%AAentry%EF%BC%8C%E9%82%A3%E4%B9%88read%E4%B8%8Edirty%E9%83%BD%E6%98%AF%E5%8F%AF%E8%A7%81%E7%9A%84%E3%80%82%0A\!%5Bf61a2a3233069db6cf8faf3dc54c3a8b.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp110\)%0Aentry.p%E6%9C%89%E4%B8%89%E7%A7%8D%E7%8A%B6%E6%80%81%EF%BC%8C%E5%88%86%E5%88%AB%E6%98%AF%EF%BC%9A%0A%0A1.%20p%20%3D%3D%20nil%0A2.%20p%20%3D%3D%20expunged%0A3.%20%E5%85%B6%E4%BB%96%0A%0A%0A\*%20\*%20\*%0A%23%23%23%23%20sync.Map%E5%8E%9F%E7%90%86%0A%0A\*%20%E9%80%9A%E8%BF%87read%E4%B8%8Edirty%E4%B8%A4%E4%B8%AA%E5%AD%97%E6%AE%B5%EF%BC%8C%E5%B0%86%E8%AF%BB%E5%86%99%E6%93%8D%E4%BD%9C%E5%88%86%E7%A6%BB%EF%BC%8C%E8%AF%BB%E7%9A%84%E6%95%B0%E6%8D%AE%E5%8F%AA%E5%AD%98%E5%9C%A8%E5%8F%AA%E8%AF%BB%E5%AD%97%E6%AE%B5read%E4%B8%8A%EF%BC%8C%E6%96%B0%E5%86%99%E5%85%A5%E7%9A%84%E6%95%B0%E6%8D%AE%E5%88%99%E6%94%BE%E5%88%B0dirty%E4%B8%AD%E3%80%82%0A\*%20%E8%AF%BB%E5%8F%96%E7%9A%84%E6%97%B6%E5%80%99%E4%BC%9A%E5%85%88%E8%AF%BBread%EF%BC%8Cread%E4%B8%AD%E6%B2%A1%E6%9C%89%E5%86%8D%E6%9F%A5%E8%AF%A2dirty%E3%80%82%0A\*%20%E8%AF%BB%E5%8F%96read%E4%B8%8D%E5%8A%A0%E9%94%81%EF%BC%8C%E4%BD%86%E6%93%8D%E4%BD%9Cdirty%E9%9C%80%E8%A6%81mutex%E4%BF%9D%E6%8A%A4%E3%80%82%0A\*%20misses%E5%AD%97%E6%AE%B5%E8%A1%A8%E7%A4%BAread%E8%AF%BB%E5%8F%96%E5%A4%B1%E8%B4%A5%E4%BD%86%E6%98%AF%E5%9C%A8dirty%E8%AF%BB%E5%88%B0%EF%BC%88read%E8%A2%AB%E7%A9%BF%E9%80%8F%EF%BC%89%E7%9A%84%E6%AC%A1%E6%95%B0%EF%BC%8C%E5%BD%93misses%E8%BE%BE%E5%88%B0%E4%B8%80%E5%AE%9A%E5%80%BC%EF%BC%8C%E9%9C%80%E8%A6%81%E5%B0%86dirty%E5%8D%87%E7%BA%A7%E4%B8%BAread%EF%BC%8C%E5%B7%B2%E5%87%8F%E5%B0%91miss%E7%9A%84%E6%AC%A1%E6%95%B0%E3%80%82%0A\*%20%E5%AF%B9%E4%BA%8E%E5%88%A0%E9%99%A4%E7%9A%84%E6%95%B0%E6%8D%AE%E9%80%9A%E8%BF%87%E6%A0%87%E8%AE%B0%E6%9D%A5%E5%BB%B6%E8%BF%9F%E5%88%A0%E9%99%A4%E3%80%82%0A%0A%23%23%23%23%20sync.Map%E6%93%8D%E4%BD%9C%0A%0A
