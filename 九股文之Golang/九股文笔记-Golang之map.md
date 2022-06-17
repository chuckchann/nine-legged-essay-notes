# 九股文笔记-Golang之map

#### 哈希表

map即哈希表，也称为散列表，是根据关键码值\(key value\)而直接进行访问的数据结构。也就是说，它通过把关键码值映射（哈希函数）到表中一个位置来访问记录，以加快查找的速度。

#### 哈希碰撞

哈希函数是整个哈希表的关键。所以为了更好的性能，我们希望在尽可能短的时间内，相同的key经过哈希函数的计算，可以得到相同的索引，不同的key经过哈希函数的计算，可以得到不同的索引，但在实际中往往事与愿违，不同的key小概率会计算出相同的索引，这就是哈希冲突（collision），几乎所有的哈希函数都存在这个问题。常见的解决哈希冲突的方法有：

1. _开放寻址法_：开放寻址法是如果通过哈希函数计算出的key所对应的空间已经被占用了，就从数组尾部再找一个还没被占用的空间将数据存进去。
2. _拉链法_：实现拉链法一般会使用数组加上链表，数组的每个索引位置装的是一个链表，当发生hash冲突时，将新的kv挂在这个链表的尾部。

#### 版本

**以下内容基于**：

```
go version go1.16.2 darwin/amd64
```

#### map的数据结构

Golang里map由runtime.hmap实现。

```

type hmap struct {
	count     int    // map的大小，指map中有多少个kv对，也就是len()的值，
	flags     uint8  // 状态标识
	B         uint8  // 哈希表中bucket的数量都2的倍数，这个字段指这个map拥有2^B个bucket(为什么要用2^B来表示而不直接用一个int来记录bucket数量？后面会介绍)
	noverflow uint16 // 溢出桶的个数
	hash0     uint32 // 哈希因子

	buckets    unsafe.Pointer // 指向当前桶数据的指针(所有桶是存放在一段连续的内存内的) 
	oldbuckets unsafe.Pointer // 在扩容之前保存的buckets字段，它的大小是当前buckets的一半
	nevacuate  uintptr        // 迁移的进度

	extra *mapextra   // 记录溢出桶的信息
}

```

hamp里的桶是由runtime.bmap表示的

```
type bmap struct {
	tophash [bucketCnt]uint8 ////一个长度为8的数组 其中每个index存储的是key的hash值的高8位 如果tophash[0] < minTopHash，则 tophash [0] 表示为迁移进度

	//因为哈希表中可能存储不同类型的键值对，当前泛形也还未大规模使用，所以键值对占据的内存空间只能通在编译的时候进行推导，下面这些字段是经过编译推导后得出的
    keys     [8]keytype     //bucket里的8个key
    values   [8]valuetype   //bucket里的8个value
    pad      uintptr
    overflow uintptr        //下一个溢出桶的指针地址
}

```

在bmap的设计中，存储kv并不是用k/v/k/v/k/v/k/v 的模式，而是 k/k/k/k/v/v/v/v 的形式去存储。这是考虑到k/v/k/v/k/v/k/v模式需要内存对齐来补存空间，会造成内存浪费。如果以k/k/k/k/v/v/v/v的形式存放，能够解决因内存对齐导致的内存浪费的问题。

一个bmap最多只能存储8个kv（即8个槽位），如果发生了哈希冲突，导致某一个桶存储的kv超过了8个，就会发生桶溢出，那么溢出的kv就会放在extra.overflow里。

![8e14228ae5dea9e82ed61345fa9ef900.png](image/8e14228ae5dea9e82ed61345fa9ef900.png)

这里先介绍下槽的几种状态，后面的代码分析中会遇到

```
const (
	//...

	emptyRest      = 0 // 表示该槽后面的槽都没有key了，并且该槽所在的桶后面的溢出桶也没有数据了
	emptyOne       = 1 // 表示槽位已经被清空
    
    //evacuatedX/evacuatedY表示桶已被迁移
	evacuatedX     = 2 // 表示该桶被迁移到序号低的桶
	evacuatedY     = 3 // 表示该桶被迁移到序号高的桶
	evacuatedEmpty = 4 // 表示操已被迁移，并且该槽对应的桶也已迁移完成
	
   
    minTopHash     = 5 // 表示最小的tophash值了，当一个key计算出来的   tophash比这个还小的话就要加上minTopHash作为最终的tophash，防止tophash跟这些标识位冲突
)
```

hmap.extra用来存放溢出桶的信息。

```
type mapextra struct {
	overflow    *[]*bmap //当前溢出桶的地址
	oldoverflow *[]*bmap //旧的溢出桶的地址
 

	nextOverflow *bmap //为空闲溢出桶的指针地址
}
```

前面我们说过，hmap.buckets指向一段内存，这段内存里放着一个正常桶数组，而extra.overflow里则指针存放的则是溢出桶数组，这两种桶在内存上是连续的。如图，黄色的为正常桶，绿色的为溢出桶。不过桶溢出只是临时的解决方案，如果有过多的桶溢出，还是会触发哈希扩容。

![11b0527c3ced1d60234bdcd61dddfa96.png](image/11b0527c3ced1d60234bdcd61dddfa96.png)

#### map的初始化

构造一个map最终都是调用runtime.makemap

```

func makemap(t *maptype, hint int, h *hmap) *hmap {
	//hint为元素个数 这里计算哈希占用的内存是否溢出或超出能分配的最大内存
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	if h == nil {
		h = new(hmap)
	}

	//初始化哈希因子
	h.hash0 = fastrand()

	//计算所需要的桶的数量
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	if h.B != 0 {
		var nextOverflow *bmap
		
		//为hmap.buckets创建保存桶的数组 
		//如果B大于4（即桶的数量大于2^4），那么将会额外为它创建 2^(B-4) 个溢出桶 
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

```

#### map的访问

map的访问主要是runtime.mapaccess1与runtime.mapaccess2，两个函数分别对应map的获取的两种方式

mapaccess1:

```
v := m[k] //mapaccess1 直接返回目标值
```

mapaccess2:

```
v, ok := m[k] //mapaccess2 除了返回目标值外 还会返回勇于判断k是否存在的bool变量
```

mapaccess1与mapaccess2整体流程基本相同，唯一不同点是mapaccess2在获取不到数据的时候会返回多一个false表示这个key不存在，这里就以mapaccess1为例来说明在map里如何根据key来获取value。

```
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {

	//省略一些前置判断...

	//如果并发读写 抛出异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}

	//对key进行hash
	hash := t.hasher(key, uintptr(h.hash0))

	//从hmap.buckets获取key对应的桶的 
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

	//是否正在扩容阶段
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
           //若正在扩容，则到老的 buckets 中查找（因为 buckets 中可能还没有值，搬迁未完成）
			b = oldb
		}
	}

	//获取hash值的高8位
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {  //先在正常桶里找 找不到就在溢出桶里找
		//根据计算出来的哈希值的高8位 依次与桶里已经存储的tophash的进行比较（快速试错，不会一开始就直接拿key来进行对比）
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest { //这个桶后面都没有kv了，直接退出当前桶的遍历
					break bucketloop
				}
				continue
			}

			//取出桶里的key
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}

			//判断桶里的key与所需要找到key是否相等
			if t.key.equal(key, k) {
				//如果相等 取出key对应的value 然后返回
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}

	//找不到 返回类型零值
	return unsafe.Pointer(&zeroVal[0])
}
```

在上面计算key对应的桶的过程中，一般来说，根据map的hash函数计算出key对应的hash值后，需要用这个hash值对hmap.buckets的长度取模（因为哈希值很可能大于bucket数组的长度，所以需要取模来确保hash值能对落到应到buckets上）。不过这里并没有采用 hash值 % bucketsize 方法计算对应的bucket，而是使用 hasn值 & bucketsize\-1 的方法来**快速计算**对应的桶的位置（也许是因为位操作比取模操作快），**但是这种位操作代替取模的前提是，bucket数组长度必须是2的指数**，这里也就解释了为什么不直接用一个int变量来存储bucketsize，而是使用hmap.B，因为bucketsize都是 2 ^ B，**所以可以用位操作来代替取模操作。**

#### map的写入

map的写入使用的是runtime.mapassign函数，该函数与runtime.mapaccess1比较类似。

```
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	
	//省略一些前置判断...

	//不允许并发写
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	
	//通过哈希函数计算key对应的哈希值
	hash := t.hasher(key, uintptr(h.hash0))

	//标记为正在写
	h.flags ^= hashWriting

	//如果hmap.buckets没有初始化 则初始化buckets
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    
	bucket := hash & bucketMask(h.B)
    
	//如果当前正在扩容阶段 那么先将扩容完成
	if h.growing() {
		growWork(t, h, bucket)
	}

	//找到桶的位置
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ { //遍历桶里的8个槽
			if b.tophash[i] != top {
				//若遍历到的槽的tophash为空, 则先记录下该位置  后面可能直接用这个位置来放置kv键值对
				if isEmpty(b.tophash[i]) && inserti == nil { 
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				
               //该槽后面都没有kv了 没有必要找了
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}

			//如果槽的tophash与参数key的tophash相等 进一步取出槽对应的key 
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}

			//判断 槽对应的key 与 入参key 是否相等
			if !t.key.equal(key, k) {
				continue
			}

			//相等 则说明map已经存在这个key了 更新key对应的value
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))

			//更新完成 直接清除写标志位然后返回
			goto done
		}

		//正常桶里没有 继续在溢出桶里找
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	//到这里说说明当前map不存在这个key
	//如果当前没有正在扩容 再判断下面两个条件：
	//1.装载因子（元素数量/桶的数量） > 6.5
	//2.map使用太多溢出桶了
	//如果满足其中任意一个条件 则进入扩容过程

	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h) //扩容
		goto again // 重新判断
	}

	//不需要扩容 判断在当前桶（正常桶）里是否能有放新的kv的位置 
	if inserti == nil {
		//正常桶里没有位置了 新生成一个溢出桶来存放 将溢出桶的第一个槽的设置为kv插入的位置
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// 将kv放入对应插入位置
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

	//清除写标志位
done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}

```

map扩容的条件有两个，分别对应两种扩容方式。

1. **如果装载因子\>6.5** map的扩容策略为将hmap.B加一, 即将整个哈希桶数目扩充为原来的两倍大小, 这种策略属于**增量扩容**。
2. **如果是溢出桶过多**，也会触发扩容。为什么太多溢出桶太，装载因子并没有超过阈值，也会需要扩容？考虑这么一个case, 向 map 中插入大量的元素, 哈希桶将逐渐被填满, 这个过程中也可能创建了一些溢出桶, 但此时装载因子并没有超过设定的阈值, 然后对这些 map 做删除操作, 删除元素之后, map 中的元素数目变少, 使得装载因子降低, 而后又重复上述的过程, 最终使得整体的装载因子不大, 但整个 map 中存在了大量的溢出桶, 因此当溢出桶数目过多时, 即便没有达到装载因子 6.5 的阈值也会触发扩容。这种扩容属于**等量扩容** 。

当满足扩容条件时，就会调用runtime.hashGrow，但runtime.hashGrow只会构造新桶并且修改相关字段，**并不会真正把数据迁移到新桶上**。**只有当map进行插入跟删除操作，才会把数据从旧桶迁移到新桶，迁移的过程是runtime.growWork完成的**。

```
func hashGrow(t *maptype, h *hmap) {
	
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow //如果装载因子不大于6.5 则为增量扩容
	}

	//保存旧桶 并构造新桶
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	
	// 相关参数修改
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	//将当前溢出桶赋值给旧溢出桶 当前溢出桶更新为nil
	if h.extra != nil && h.extra.overflow != nil {
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}

	//重新赋值nextOverflow
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

}
```

下面是触发扩容后的哈希。

![3a7e3be5ecc4fa77b61ac6166ca23ab3.png](image/3a7e3be5ecc4fa77b61ac6166ca23ab3.png)

**runtime.growWork**负责出处理桶迁移，而其中又调用**runtime.evacuate**来具体迁移某个桶，可以看到一次迁移过程最多搬移**两个桶**，而不是一次性将所有旧桶都迁移到新桶，这样做是为了**避免一次迁移太桶多会造成区间性能抖动**。

```
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 当前操作key对应的桶（旧桶），搬迁到新桶数组
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

**runtime.evacuate**迁移流程如下。

```
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	//找到key对应的旧桶
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	// newbit即为老桶的个数
	newbit := h.noldbuckets()

	if !evacuated(b) {
		// xy 是把老桶迁移到新桶去的位置，x是低位，y是高位
		var xy [2]evacDst
		x := &xy[0]
		// x和老桶的位置相同（例如在老桶数组中位置数是3 那么迁移后在新桶数组中也是3）
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		// 新桶内k的起始位置
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		// 新桶内v的起始位置
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			//增量扩容 设置y x与y之间隔了oldbuckets的地址（例如有4个桶，本次迁移的是第三个老桶，那么x为3，y为3加上老桶个数4，即为7）
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) { //遍历老桶及其溢出桶
			// 老桶内k/v的起始位置
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) { //遍历老桶内的槽
				top := b.tophash[i]
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty //槽位置为空，标记为槽已迁移
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}

				
				var useY uint8
				if !h.sameSizeGrow() {
					//如果是增量扩容 需要判断key是要放在x还是放在y
					hash := t.hasher(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.equal(k2, k2) {
						// 如果满足这一条件（具体啥条件不是很能看明白。。。） 直接迁移到y
						useY = top & 1
						top = tophash(hash)
					} else {
						// 根据hash&newbit 来判断当前key是放在x还是y
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				// 标记这个槽是被迁移到x还是y
				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination
 
 				// 新桶的槽位满了，需要给新桶创建溢出桶
				if dst.i == bucketCnt {
					// 迁移目的地修改为溢出桶的信息
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}

				//设置新桶槽位的tophash
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check

				//把老桶的kv复制到新桶
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.elem, dst.e, e)
				}
				//新桶内迁移的数加一
				dst.i++

				//新桶的kv向前偏移一个单位
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		
		// 把老桶释放掉  
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))

			// 只清理kv的内存，而不清理8个槽的tophash，这里留着tophash主要是利用啊他们的标记位，已方便后续的其他的桶迁移
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	//迁移标记
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```

迁移位置示意图。

![6c9038a0a055fe082a830d7d54cc3302.png](image/6c9038a0a055fe082a830d7d54cc3302.png)

#### map的删除

从map中删除元素使用的是runtime.mapdelete函数簇中的一个，包括 runtime.mapdelete、mapdelete\_faststr、mapdelete\_fast32 和 mapdelete\_fast64，主体流程与key的插入类似，这里挑选runtime.mapdelete来分析一下。

```
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {

	//省略一些前置判断...

	//计算key的哈希值
	hash := t.hasher(key, uintptr(h.hash0))

	h.flags ^= hashWriting

	//得key对应的桶在数组中的位置
	bucket := hash & bucketMask(h.B)

	//如果正常扩容 则先搬迁两个桶
	if h.growing() {
		growWork(t, h, bucket)
	}

	//得到具体的桶
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b //存一下 后面可能会用到
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {  //遍历正常桶及溢出桶
		for i := uintptr(0); i < bucketCnt; i++ { //遍历桶的每个槽
			if b.tophash[i] != top { //用哈希值的高8位来快速试错 
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}

			//哈希值高8位相等 从槽里取出key 跟 传入的key做对比
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			
			//清除key的操作
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil 
			} else if t.key.ptrdata != 0 {  //如果key含有指针 清除key
				memclrHasPointers(k, t.key.size) 
			}

			//清除value的操作
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 { //如果value含有指针 清除value
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}

			//这个槽标记为被清除过
			b.tophash[i] = emptyOne

			//下面这一段是去判断要不要将当前的槽标记为emptyRest
			if i == bucketCnt-1 {
				//如果是桶的最后一个槽，那么判断溢出桶的第一个槽位是不是emptyRest，如果是那这个槽也将标记为emptyRest
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				//如果不是桶的最后一个槽，判断该桶的下一个槽位是不是emptyRest，如果是那这个槽也将标记为emptyRest
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}

			//到这里时需要标记当前槽为emptyRest 即该槽后面的槽都没有key了，并且该槽所在的桶后面的溢出桶也没有数据了
			//并且一直往前查找可以标记为emptyRest的槽
			for {
				b.tophash[i] = emptyRest

				//这段if/else是找当前槽的前一个位置
				if i == 0 { //当前槽位是该桶的第一个槽位 需要找前一个桶的倒数第一个槽位
					if b == bOrig {
						break //当前是正常桶的第一个槽位 没有前一个位置了 
					}
					
					//找该桶的前一个桶 因为正常桶跟溢出桶之间是单链表，所以没法直接通过当前桶找到上一个桶，只能从正常桶bOrig开始从头遍历
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else { //当前槽位不是该桶的第一个槽位 继续往前找上一个槽位
					i--
				}

				//如果前一个位置不是空槽（emptyOne）那标记emptyRest的工作就结束了
				if b.tophash[i] != emptyOne {
					break
				}
			}

		notLast:
			//元素数减少
			h.count--
			//如果map没有kv了 则重置这个map的哈希因子
			if h.count == 0 {
				h.hash0 = fastrand()
			}
			break search
		}
	}

	//清理写标志位
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

#### map的遍历

如果单纯地遍历map是一件比较简单的事，最外层遍历所有的bucket\(包括oldbuckets\)，中间遍历bucket里的槽\(包括溢出桶\)，即可获取map里的所有的kv对。但实际上map的遍历并不是有序的，Go团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

在遍历map时，编译器会使用 runtime.mapiterinit 和 runtime.mapiternext 两个运行时函数重写原始的 for\-range 循环。

runtime.mapiterinit会初始化遍历开始的元素。其核心逻辑如下。

```
runtime.mapiterinit的核心逻辑如下。
func mapiterinit(t *maptype, h *hmap, it *hiter) {

	...

	it.t = t
	it.h = h
	it.B = h.B
	it.buckets = h.buckets

	if t.bucket.ptrdata == 0 {
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// 随机因子
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	// 根据随机因子计算 起始位置
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	mapiternext(it)
}
```

runtime.mapiternext负责遍历元素。其中又分为选择桶与遍历桶内元素两个阶段。其核心逻辑如下：

```
func mapiternext(it *hiter) {
	h := it.h
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			it.key = nil
			it.elem = nil
			return
		}

		//如何桶为空 找需要遍历的新桶
		if h.growing() && it.B == h.B {
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}

	//剩下的部分就是遍历桶的元素，不过如果哈希表处于扩容期间就会调用 runtime.mapaccessK 获取键值对：
		for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			it.key = k
			it.value = v
		} else {
			rk, rv := mapaccessK(t, h, k)
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		it.i = i + 1
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}

```

%23%23%23%23%20%E5%93%88%E5%B8%8C%E8%A1%A8%0Amap%E5%8D%B3%E5%93%88%E5%B8%8C%E8%A1%A8%EF%BC%8C%E4%B9%9F%E7%A7%B0%E4%B8%BA%E6%95%A3%E5%88%97%E8%A1%A8%EF%BC%8C%E6%98%AF%E6%A0%B9%E6%8D%AE%E5%85%B3%E9%94%AE%E7%A0%81%E5%80%BC\(key%20value\)%E8%80%8C%E7%9B%B4%E6%8E%A5%E8%BF%9B%E8%A1%8C%E8%AE%BF%E9%97%AE%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E3%80%82%E4%B9%9F%E5%B0%B1%E6%98%AF%E8%AF%B4%EF%BC%8C%E5%AE%83%E9%80%9A%E8%BF%87%E6%8A%8A%E5%85%B3%E9%94%AE%E7%A0%81%E5%80%BC%E6%98%A0%E5%B0%84%EF%BC%88%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%EF%BC%89%E5%88%B0%E8%A1%A8%E4%B8%AD%E4%B8%80%E4%B8%AA%E4%BD%8D%E7%BD%AE%E6%9D%A5%E8%AE%BF%E9%97%AE%E8%AE%B0%E5%BD%95%EF%BC%8C%E4%BB%A5%E5%8A%A0%E5%BF%AB%E6%9F%A5%E6%89%BE%E7%9A%84%E9%80%9F%E5%BA%A6%E3%80%82%0A%0A%23%23%23%23%20%E5%93%88%E5%B8%8C%E7%A2%B0%E6%92%9E%0A%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E6%98%AF%E6%95%B4%E4%B8%AA%E5%93%88%E5%B8%8C%E8%A1%A8%E7%9A%84%E5%85%B3%E9%94%AE%E3%80%82%E6%89%80%E4%BB%A5%E4%B8%BA%E4%BA%86%E6%9B%B4%E5%A5%BD%E7%9A%84%E6%80%A7%E8%83%BD%EF%BC%8C%E6%88%91%E4%BB%AC%E5%B8%8C%E6%9C%9B%E5%9C%A8%E5%B0%BD%E5%8F%AF%E8%83%BD%E7%9F%AD%E7%9A%84%E6%97%B6%E9%97%B4%E5%86%85%EF%BC%8C%E7%9B%B8%E5%90%8C%E7%9A%84key%E7%BB%8F%E8%BF%87%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E7%9A%84%E8%AE%A1%E7%AE%97%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%BE%97%E5%88%B0%E7%9B%B8%E5%90%8C%E7%9A%84%E7%B4%A2%E5%BC%95%EF%BC%8C%E4%B8%8D%E5%90%8C%E7%9A%84key%E7%BB%8F%E8%BF%87%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E7%9A%84%E8%AE%A1%E7%AE%97%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%BE%97%E5%88%B0%E4%B8%8D%E5%90%8C%E7%9A%84%E7%B4%A2%E5%BC%95%EF%BC%8C%E4%BD%86%E5%9C%A8%E5%AE%9E%E9%99%85%E4%B8%AD%E5%BE%80%E5%BE%80%E4%BA%8B%E4%B8%8E%E6%84%BF%E8%BF%9D%EF%BC%8C%E4%B8%8D%E5%90%8C%E7%9A%84key%E5%B0%8F%E6%A6%82%E7%8E%87%E4%BC%9A%E8%AE%A1%E7%AE%97%E5%87%BA%E7%9B%B8%E5%90%8C%E7%9A%84%E7%B4%A2%E5%BC%95%EF%BC%8C%E8%BF%99%E5%B0%B1%E6%98%AF%E5%93%88%E5%B8%8C%E5%86%B2%E7%AA%81%EF%BC%88collision%EF%BC%89%EF%BC%8C%E5%87%A0%E4%B9%8E%E6%89%80%E6%9C%89%E7%9A%84%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E9%83%BD%E5%AD%98%E5%9C%A8%E8%BF%99%E4%B8%AA%E9%97%AE%E9%A2%98%E3%80%82%E5%B8%B8%E8%A7%81%E7%9A%84%E8%A7%A3%E5%86%B3%E5%93%88%E5%B8%8C%E5%86%B2%E7%AA%81%E7%9A%84%E6%96%B9%E6%B3%95%E6%9C%89%EF%BC%9A%0A%0A1.%20%3Cu%3E\*%E5%BC%80%E6%94%BE%E5%AF%BB%E5%9D%80%E6%B3%95\*%3C%2Fu%3E%EF%BC%9A%E5%BC%80%E6%94%BE%E5%AF%BB%E5%9D%80%E6%B3%95%E6%98%AF%E5%A6%82%E6%9E%9C%E9%80%9A%E8%BF%87%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97%E5%87%BA%E7%9A%84key%E6%89%80%E5%AF%B9%E5%BA%94%E7%9A%84%E7%A9%BA%E9%97%B4%E5%B7%B2%E7%BB%8F%E8%A2%AB%E5%8D%A0%E7%94%A8%E4%BA%86%EF%BC%8C%E5%B0%B1%E4%BB%8E%E6%95%B0%E7%BB%84%E5%B0%BE%E9%83%A8%E5%86%8D%E6%89%BE%E4%B8%80%E4%B8%AA%E8%BF%98%E6%B2%A1%E8%A2%AB%E5%8D%A0%E7%94%A8%E7%9A%84%E7%A9%BA%E9%97%B4%E5%B0%86%E6%95%B0%E6%8D%AE%E5%AD%98%E8%BF%9B%E5%8E%BB%E3%80%82%0A2.%20%3Cu%3E\*%E6%8B%89%E9%93%BE%E6%B3%95\*%3C%2Fu%3E%EF%BC%9A%E5%AE%9E%E7%8E%B0%E6%8B%89%E9%93%BE%E6%B3%95%E4%B8%80%E8%88%AC%E4%BC%9A%E4%BD%BF%E7%94%A8%E6%95%B0%E7%BB%84%E5%8A%A0%E4%B8%8A%E9%93%BE%E8%A1%A8%EF%BC%8C%E6%95%B0%E7%BB%84%E7%9A%84%E6%AF%8F%E4%B8%AA%E7%B4%A2%E5%BC%95%E4%BD%8D%E7%BD%AE%E8%A3%85%E7%9A%84%E6%98%AF%E4%B8%80%E4%B8%AA%E9%93%BE%E8%A1%A8%EF%BC%8C%E5%BD%93%E5%8F%91%E7%94%9Fhash%E5%86%B2%E7%AA%81%E6%97%B6%EF%BC%8C%E5%B0%86%E6%96%B0%E7%9A%84kv%E6%8C%82%E5%9C%A8%E8%BF%99%E4%B8%AA%E9%93%BE%E8%A1%A8%E7%9A%84%E5%B0%BE%E9%83%A8%E3%80%82%0A%0A%23%23%23%23%20%E7%89%88%E6%9C%AC%0A%0A\*\*%E4%BB%A5%E4%B8%8B%E5%86%85%E5%AE%B9%E5%9F%BA%E4%BA%8E\*\*%EF%BC%9A%0A%0A%60%60%60%0Ago%20version%20go1.16.2%20darwin%2Famd64%0A%60%60%60%0A%0A%23%23%23%23%20map%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%0AGolang%E9%87%8Cmap%E7%94%B1runtime.hmap%E5%AE%9E%E7%8E%B0%E3%80%82%0A%60%60%60%0A%0Atype%20hmap%20struct%20%7B%0A%09count%20%20%20%20%20int%20%20%20%20%2F%2F%20map%E7%9A%84%E5%A4%A7%E5%B0%8F%EF%BC%8C%E6%8C%87map%E4%B8%AD%E6%9C%89%E5%A4%9A%E5%B0%91%E4%B8%AAkv%E5%AF%B9%EF%BC%8C%E4%B9%9F%E5%B0%B1%E6%98%AFlen\(\)%E7%9A%84%E5%80%BC%EF%BC%8C%0A%09flags%20%20%20%20%20uint8%20%20%2F%2F%20%E7%8A%B6%E6%80%81%E6%A0%87%E8%AF%86%0A%09B%20%20%20%20%20%20%20%20%20uint8%20%20%2F%2F%20%E5%93%88%E5%B8%8C%E8%A1%A8%E4%B8%ADbucket%E7%9A%84%E6%95%B0%E9%87%8F%E9%83%BD2%E7%9A%84%E5%80%8D%E6%95%B0%EF%BC%8C%E8%BF%99%E4%B8%AA%E5%AD%97%E6%AE%B5%E6%8C%87%E8%BF%99%E4%B8%AAmap%E6%8B%A5%E6%9C%892%5EB%E4%B8%AAbucket\(%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E7%94%A82%5EB%E6%9D%A5%E8%A1%A8%E7%A4%BA%E8%80%8C%E4%B8%8D%E7%9B%B4%E6%8E%A5%E7%94%A8%E4%B8%80%E4%B8%AAint%E6%9D%A5%E8%AE%B0%E5%BD%95bucket%E6%95%B0%E9%87%8F%EF%BC%9F%E5%90%8E%E9%9D%A2%E4%BC%9A%E4%BB%8B%E7%BB%8D\)%0A%09noverflow%20uint16%20%2F%2F%20%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E4%B8%AA%E6%95%B0%0A%09hash0%20%20%20%20%20uint32%20%2F%2F%20%E5%93%88%E5%B8%8C%E5%9B%A0%E5%AD%90%0A%0A%09buckets%20%20%20%20unsafe.Pointer%20%2F%2F%20%E6%8C%87%E5%90%91%E5%BD%93%E5%89%8D%E6%A1%B6%E6%95%B0%E6%8D%AE%E7%9A%84%E6%8C%87%E9%92%88\(%E6%89%80%E6%9C%89%E6%A1%B6%E6%98%AF%E5%AD%98%E6%94%BE%E5%9C%A8%E4%B8%80%E6%AE%B5%E8%BF%9E%E7%BB%AD%E7%9A%84%E5%86%85%E5%AD%98%E5%86%85%E7%9A%84\)%20%0A%09oldbuckets%20unsafe.Pointer%20%2F%2F%20%E5%9C%A8%E6%89%A9%E5%AE%B9%E4%B9%8B%E5%89%8D%E4%BF%9D%E5%AD%98%E7%9A%84buckets%E5%AD%97%E6%AE%B5%EF%BC%8C%E5%AE%83%E7%9A%84%E5%A4%A7%E5%B0%8F%E6%98%AF%E5%BD%93%E5%89%8Dbuckets%E7%9A%84%E4%B8%80%E5%8D%8A%0A%09nevacuate%20%20uintptr%20%20%20%20%20%20%20%20%2F%2F%20%E8%BF%81%E7%A7%BB%E7%9A%84%E8%BF%9B%E5%BA%A6%0A%0A%09extra%20\*mapextra%20%20%20%2F%2F%20%E8%AE%B0%E5%BD%95%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E4%BF%A1%E6%81%AF%0A%7D%0A%0A%60%60%60%0A%0Ahamp%E9%87%8C%E7%9A%84%E6%A1%B6%E6%98%AF%E7%94%B1runtime.bmap%E8%A1%A8%E7%A4%BA%E7%9A%84%0A%0A%60%60%60%0Atype%20bmap%20struct%20%7B%0A%09tophash%20%5BbucketCnt%5Duint8%20%2F%2F%2F%2F%E4%B8%80%E4%B8%AA%E9%95%BF%E5%BA%A6%E4%B8%BA8%E7%9A%84%E6%95%B0%E7%BB%84%20%E5%85%B6%E4%B8%AD%E6%AF%8F%E4%B8%AAindex%E5%AD%98%E5%82%A8%E7%9A%84%E6%98%AFkey%E7%9A%84hash%E5%80%BC%E7%9A%84%E9%AB%988%E4%BD%8D%20%E5%A6%82%E6%9E%9Ctophash%5B0%5D%20%3C%20minTopHash%EF%BC%8C%E5%88%99%20tophash%20%5B0%5D%20%E8%A1%A8%E7%A4%BA%E4%B8%BA%E8%BF%81%E7%A7%BB%E8%BF%9B%E5%BA%A6%0A%0A%09%2F%2F%E5%9B%A0%E4%B8%BA%E5%93%88%E5%B8%8C%E8%A1%A8%E4%B8%AD%E5%8F%AF%E8%83%BD%E5%AD%98%E5%82%A8%E4%B8%8D%E5%90%8C%E7%B1%BB%E5%9E%8B%E7%9A%84%E9%94%AE%E5%80%BC%E5%AF%B9%EF%BC%8C%E5%BD%93%E5%89%8D%E6%B3%9B%E5%BD%A2%E4%B9%9F%E8%BF%98%E6%9C%AA%E5%A4%A7%E8%A7%84%E6%A8%A1%E4%BD%BF%E7%94%A8%EF%BC%8C%E6%89%80%E4%BB%A5%E9%94%AE%E5%80%BC%E5%AF%B9%E5%8D%A0%E6%8D%AE%E7%9A%84%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E5%8F%AA%E8%83%BD%E9%80%9A%E5%9C%A8%E7%BC%96%E8%AF%91%E7%9A%84%E6%97%B6%E5%80%99%E8%BF%9B%E8%A1%8C%E6%8E%A8%E5%AF%BC%EF%BC%8C%E4%B8%8B%E9%9D%A2%E8%BF%99%E4%BA%9B%E5%AD%97%E6%AE%B5%E6%98%AF%E7%BB%8F%E8%BF%87%E7%BC%96%E8%AF%91%E6%8E%A8%E5%AF%BC%E5%90%8E%E5%BE%97%E5%87%BA%E7%9A%84%0A%20%20%20%20keys%20%20%20%20%20%5B8%5Dkeytype%20%20%20%20%20%2F%2Fbucket%E9%87%8C%E7%9A%848%E4%B8%AAkey%0A%20%20%20%20values%20%20%20%5B8%5Dvaluetype%20%20%20%2F%2Fbucket%E9%87%8C%E7%9A%848%E4%B8%AAvalue%0A%20%20%20%20pad%20%20%20%20%20%20uintptr%0A%20%20%20%20overflow%20uintptr%20%20%20%20%20%20%20%20%2F%2F%E4%B8%8B%E4%B8%80%E4%B8%AA%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E6%8C%87%E9%92%88%E5%9C%B0%E5%9D%80%0A%7D%0A%0A%60%60%60%0A%E5%9C%A8bmap%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%AD%EF%BC%8C%E5%AD%98%E5%82%A8kv%E5%B9%B6%E4%B8%8D%E6%98%AF%E7%94%A8k%2Fv%2Fk%2Fv%2Fk%2Fv%2Fk%2Fv%20%E7%9A%84%E6%A8%A1%E5%BC%8F%EF%BC%8C%E8%80%8C%E6%98%AF%20k%2Fk%2Fk%2Fk%2Fv%2Fv%2Fv%2Fv%20%E7%9A%84%E5%BD%A2%E5%BC%8F%E5%8E%BB%E5%AD%98%E5%82%A8%E3%80%82%E8%BF%99%E6%98%AF%E8%80%83%E8%99%91%E5%88%B0k%2Fv%2Fk%2Fv%2Fk%2Fv%2Fk%2Fv%E6%A8%A1%E5%BC%8F%E9%9C%80%E8%A6%81%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90%E6%9D%A5%E8%A1%A5%E5%AD%98%E7%A9%BA%E9%97%B4%EF%BC%8C%E4%BC%9A%E9%80%A0%E6%88%90%E5%86%85%E5%AD%98%E6%B5%AA%E8%B4%B9%E3%80%82%E5%A6%82%E6%9E%9C%E4%BB%A5k%2Fk%2Fk%2Fk%2Fv%2Fv%2Fv%2Fv%E7%9A%84%E5%BD%A2%E5%BC%8F%E5%AD%98%E6%94%BE%EF%BC%8C%E8%83%BD%E5%A4%9F%E8%A7%A3%E5%86%B3%E5%9B%A0%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90%E5%AF%BC%E8%87%B4%E7%9A%84%E5%86%85%E5%AD%98%E6%B5%AA%E8%B4%B9%E7%9A%84%E9%97%AE%E9%A2%98%E3%80%82%0A%0A%E4%B8%80%E4%B8%AAbmap%E6%9C%80%E5%A4%9A%E5%8F%AA%E8%83%BD%E5%AD%98%E5%82%A88%E4%B8%AAkv%EF%BC%88%E5%8D%B38%E4%B8%AA%E6%A7%BD%E4%BD%8D%EF%BC%89%EF%BC%8C%E5%A6%82%E6%9E%9C%E5%8F%91%E7%94%9F%E4%BA%86%E5%93%88%E5%B8%8C%E5%86%B2%E7%AA%81%EF%BC%8C%E5%AF%BC%E8%87%B4%E6%9F%90%E4%B8%80%E4%B8%AA%E6%A1%B6%E5%AD%98%E5%82%A8%E7%9A%84kv%E8%B6%85%E8%BF%87%E4%BA%868%E4%B8%AA%EF%BC%8C%E5%B0%B1%E4%BC%9A%E5%8F%91%E7%94%9F%E6%A1%B6%E6%BA%A2%E5%87%BA%EF%BC%8C%E9%82%A3%E4%B9%88%E6%BA%A2%E5%87%BA%E7%9A%84kv%E5%B0%B1%E4%BC%9A%E6%94%BE%E5%9C%A8extra.overflow%E9%87%8C%E3%80%82%0A%0A\!%5B8e14228ae5dea9e82ed61345fa9ef900.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp100\)%0A%0A%E8%BF%99%E9%87%8C%E5%85%88%E4%BB%8B%E7%BB%8D%E4%B8%8B%E6%A7%BD%E7%9A%84%E5%87%A0%E7%A7%8D%E7%8A%B6%E6%80%81%EF%BC%8C%E5%90%8E%E9%9D%A2%E7%9A%84%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%AD%E4%BC%9A%E9%81%87%E5%88%B0%0A%60%60%60%0Aconst%20\(%0A%09%2F%2F...%0A%0A%09emptyRest%20%20%20%20%20%20%3D%200%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E8%AF%A5%E6%A7%BD%E5%90%8E%E9%9D%A2%E7%9A%84%E6%A7%BD%E9%83%BD%E6%B2%A1%E6%9C%89key%E4%BA%86%EF%BC%8C%E5%B9%B6%E4%B8%94%E8%AF%A5%E6%A7%BD%E6%89%80%E5%9C%A8%E7%9A%84%E6%A1%B6%E5%90%8E%E9%9D%A2%E7%9A%84%E6%BA%A2%E5%87%BA%E6%A1%B6%E4%B9%9F%E6%B2%A1%E6%9C%89%E6%95%B0%E6%8D%AE%E4%BA%86%0A%09emptyOne%20%20%20%20%20%20%20%3D%201%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E6%A7%BD%E4%BD%8D%E5%B7%B2%E7%BB%8F%E8%A2%AB%E6%B8%85%E7%A9%BA%0A%20%20%20%20%0A%20%20%20%20%2F%2FevacuatedX%2FevacuatedY%E8%A1%A8%E7%A4%BA%E6%A1%B6%E5%B7%B2%E8%A2%AB%E8%BF%81%E7%A7%BB%0A%09evacuatedX%20%20%20%20%20%3D%202%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E8%AF%A5%E6%A1%B6%E8%A2%AB%E8%BF%81%E7%A7%BB%E5%88%B0%E5%BA%8F%E5%8F%B7%E4%BD%8E%E7%9A%84%E6%A1%B6%0A%09evacuatedY%20%20%20%20%20%3D%203%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E8%AF%A5%E6%A1%B6%E8%A2%AB%E8%BF%81%E7%A7%BB%E5%88%B0%E5%BA%8F%E5%8F%B7%E9%AB%98%E7%9A%84%E6%A1%B6%0A%09evacuatedEmpty%20%3D%204%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E6%93%8D%E5%B7%B2%E8%A2%AB%E8%BF%81%E7%A7%BB%EF%BC%8C%E5%B9%B6%E4%B8%94%E8%AF%A5%E6%A7%BD%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%E4%B9%9F%E5%B7%B2%E8%BF%81%E7%A7%BB%E5%AE%8C%E6%88%90%0A%09%0A%20%20%20%0A%20%20%20%20minTopHash%20%20%20%20%20%3D%205%20%2F%2F%20%E8%A1%A8%E7%A4%BA%E6%9C%80%E5%B0%8F%E7%9A%84tophash%E5%80%BC%E4%BA%86%EF%BC%8C%E5%BD%93%E4%B8%80%E4%B8%AAkey%E8%AE%A1%E7%AE%97%E5%87%BA%E6%9D%A5%E7%9A%84%20%20%20tophash%E6%AF%94%E8%BF%99%E4%B8%AA%E8%BF%98%E5%B0%8F%E7%9A%84%E8%AF%9D%E5%B0%B1%E8%A6%81%E5%8A%A0%E4%B8%8AminTopHash%E4%BD%9C%E4%B8%BA%E6%9C%80%E7%BB%88%E7%9A%84tophash%EF%BC%8C%E9%98%B2%E6%AD%A2tophash%E8%B7%9F%E8%BF%99%E4%BA%9B%E6%A0%87%E8%AF%86%E4%BD%8D%E5%86%B2%E7%AA%81%0A\)%0A%60%60%60%0A%0Ahmap.extra%E7%94%A8%E6%9D%A5%E5%AD%98%E6%94%BE%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E4%BF%A1%E6%81%AF%E3%80%82%0A%60%60%60%0Atype%20mapextra%20struct%20%7B%0A%09overflow%20%20%20%20\*%5B%5D\*bmap%20%2F%2F%E5%BD%93%E5%89%8D%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E5%9C%B0%E5%9D%80%0A%09oldoverflow%20\*%5B%5D\*bmap%20%2F%2F%E6%97%A7%E7%9A%84%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E5%9C%B0%E5%9D%80%0A%20%0A%0A%09nextOverflow%20\*bmap%20%2F%2F%E4%B8%BA%E7%A9%BA%E9%97%B2%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E6%8C%87%E9%92%88%E5%9C%B0%E5%9D%80%0A%7D%0A%60%60%60%0A%0A%E5%89%8D%E9%9D%A2%E6%88%91%E4%BB%AC%E8%AF%B4%E8%BF%87%EF%BC%8Chmap.buckets%E6%8C%87%E5%90%91%E4%B8%80%E6%AE%B5%E5%86%85%E5%AD%98%EF%BC%8C%E8%BF%99%E6%AE%B5%E5%86%85%E5%AD%98%E9%87%8C%E6%94%BE%E7%9D%80%E4%B8%80%E4%B8%AA%E6%AD%A3%E5%B8%B8%E6%A1%B6%E6%95%B0%E7%BB%84%EF%BC%8C%E8%80%8Cextra.overflow%E9%87%8C%E5%88%99%E6%8C%87%E9%92%88%E5%AD%98%E6%94%BE%E7%9A%84%E5%88%99%E6%98%AF%E6%BA%A2%E5%87%BA%E6%A1%B6%E6%95%B0%E7%BB%84%EF%BC%8C%E8%BF%99%E4%B8%A4%E7%A7%8D%E6%A1%B6%E5%9C%A8%E5%86%85%E5%AD%98%E4%B8%8A%E6%98%AF%E8%BF%9E%E7%BB%AD%E7%9A%84%E3%80%82%E5%A6%82%E5%9B%BE%EF%BC%8C%E9%BB%84%E8%89%B2%E7%9A%84%E4%B8%BA%E6%AD%A3%E5%B8%B8%E6%A1%B6%EF%BC%8C%E7%BB%BF%E8%89%B2%E7%9A%84%E4%B8%BA%E6%BA%A2%E5%87%BA%E6%A1%B6%E3%80%82%E4%B8%8D%E8%BF%87%E6%A1%B6%E6%BA%A2%E5%87%BA%E5%8F%AA%E6%98%AF%E4%B8%B4%E6%97%B6%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%8C%E5%A6%82%E6%9E%9C%E6%9C%89%E8%BF%87%E5%A4%9A%E7%9A%84%E6%A1%B6%E6%BA%A2%E5%87%BA%EF%BC%8C%E8%BF%98%E6%98%AF%E4%BC%9A%E8%A7%A6%E5%8F%91%E5%93%88%E5%B8%8C%E6%89%A9%E5%AE%B9%E3%80%82%0A\!%5B11b0527c3ced1d60234bdcd61dddfa96.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp99\)%0A%0A%0A%0A%0A%0A%23%23%23%23%20map%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%0A%E6%9E%84%E9%80%A0%E4%B8%80%E4%B8%AAmap%E6%9C%80%E7%BB%88%E9%83%BD%E6%98%AF%E8%B0%83%E7%94%A8runtime.makemap%0A%0A%60%60%60%0A%0Afunc%20makemap\(t%20\*maptype%2C%20hint%20int%2C%20h%20\*hmap\)%20\*hmap%20%7B%0A%09%2F%2Fhint%E4%B8%BA%E5%85%83%E7%B4%A0%E4%B8%AA%E6%95%B0%20%E8%BF%99%E9%87%8C%E8%AE%A1%E7%AE%97%E5%93%88%E5%B8%8C%E5%8D%A0%E7%94%A8%E7%9A%84%E5%86%85%E5%AD%98%E6%98%AF%E5%90%A6%E6%BA%A2%E5%87%BA%E6%88%96%E8%B6%85%E5%87%BA%E8%83%BD%E5%88%86%E9%85%8D%E7%9A%84%E6%9C%80%E5%A4%A7%E5%86%85%E5%AD%98%0A%09mem%2C%20overflow%20%3A%3D%20math.MulUintptr\(uintptr\(hint\)%2C%20t.bucket.size\)%0A%09if%20overflow%20%7C%7C%20mem%20%3E%20maxAlloc%20%7B%0A%09%09hint%20%3D%200%0A%09%7D%0A%0A%09if%20h%20%3D%3D%20nil%20%7B%0A%09%09h%20%3D%20new\(hmap\)%0A%09%7D%0A%0A%09%2F%2F%E5%88%9D%E5%A7%8B%E5%8C%96%E5%93%88%E5%B8%8C%E5%9B%A0%E5%AD%90%0A%09h.hash0%20%3D%20fastrand\(\)%0A%0A%09%2F%2F%E8%AE%A1%E7%AE%97%E6%89%80%E9%9C%80%E8%A6%81%E7%9A%84%E6%A1%B6%E7%9A%84%E6%95%B0%E9%87%8F%0A%09B%20%3A%3D%20uint8\(0\)%0A%09for%20overLoadFactor\(hint%2C%20B\)%20%7B%0A%09%09B%2B%2B%0A%09%7D%0A%09h.B%20%3D%20B%0A%0A%09if%20h.B%20\!%3D%200%20%7B%0A%09%09var%20nextOverflow%20\*bmap%0A%09%09%0A%09%09%2F%2F%E4%B8%BAhmap.buckets%E5%88%9B%E5%BB%BA%E4%BF%9D%E5%AD%98%E6%A1%B6%E7%9A%84%E6%95%B0%E7%BB%84%20%0A%09%09%2F%2F%E5%A6%82%E6%9E%9CB%E5%A4%A7%E4%BA%8E4%EF%BC%88%E5%8D%B3%E6%A1%B6%E7%9A%84%E6%95%B0%E9%87%8F%E5%A4%A7%E4%BA%8E2%5E4%EF%BC%89%EF%BC%8C%E9%82%A3%E4%B9%88%E5%B0%86%E4%BC%9A%E9%A2%9D%E5%A4%96%E4%B8%BA%E5%AE%83%E5%88%9B%E5%BB%BA%202%5E\(B\-4\)%20%E4%B8%AA%E6%BA%A2%E5%87%BA%E6%A1%B6%20%0A%09%09h.buckets%2C%20nextOverflow%20%3D%20makeBucketArray\(t%2C%20h.B%2C%20nil\)%0A%09%09if%20nextOverflow%20\!%3D%20nil%20%7B%0A%09%09%09h.extra%20%3D%20new\(mapextra\)%0A%09%09%09h.extra.nextOverflow%20%3D%20nextOverflow%0A%09%09%7D%0A%09%7D%0A%0A%09return%20h%0A%7D%0A%0A%0A%60%60%60%0A%23%23%23%23%20map%E7%9A%84%E8%AE%BF%E9%97%AE%0Amap%E7%9A%84%E8%AE%BF%E9%97%AE%E4%B8%BB%E8%A6%81%E6%98%AFruntime.mapaccess1%E4%B8%8Eruntime.mapaccess2%EF%BC%8C%E4%B8%A4%E4%B8%AA%E5%87%BD%E6%95%B0%E5%88%86%E5%88%AB%E5%AF%B9%E5%BA%94map%E7%9A%84%E8%8E%B7%E5%8F%96%E7%9A%84%E4%B8%A4%E7%A7%8D%E6%96%B9%E5%BC%8F%0A%0Amapaccess1%3A%0A%60%60%60%0Av%20%3A%3D%20m%5Bk%5D%20%2F%2Fmapaccess1%20%E7%9B%B4%E6%8E%A5%E8%BF%94%E5%9B%9E%E7%9B%AE%E6%A0%87%E5%80%BC%0A%60%60%60%0A%0Amapaccess2%3A%0A%60%60%60%0Av%2C%20ok%20%3A%3D%20m%5Bk%5D%20%2F%2Fmapaccess2%20%E9%99%A4%E4%BA%86%E8%BF%94%E5%9B%9E%E7%9B%AE%E6%A0%87%E5%80%BC%E5%A4%96%20%E8%BF%98%E4%BC%9A%E8%BF%94%E5%9B%9E%E5%8B%87%E4%BA%8E%E5%88%A4%E6%96%ADk%E6%98%AF%E5%90%A6%E5%AD%98%E5%9C%A8%E7%9A%84bool%E5%8F%98%E9%87%8F%0A%60%60%60%0A%0A%0Amapaccess1%E4%B8%8Emapaccess2%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B%E5%9F%BA%E6%9C%AC%E7%9B%B8%E5%90%8C%EF%BC%8C%E5%94%AF%E4%B8%80%E4%B8%8D%E5%90%8C%E7%82%B9%E6%98%AFmapaccess2%E5%9C%A8%E8%8E%B7%E5%8F%96%E4%B8%8D%E5%88%B0%E6%95%B0%E6%8D%AE%E7%9A%84%E6%97%B6%E5%80%99%E4%BC%9A%E8%BF%94%E5%9B%9E%E5%A4%9A%E4%B8%80%E4%B8%AAfalse%E8%A1%A8%E7%A4%BA%E8%BF%99%E4%B8%AAkey%E4%B8%8D%E5%AD%98%E5%9C%A8%EF%BC%8C%E8%BF%99%E9%87%8C%E5%B0%B1%E4%BB%A5mapaccess1%E4%B8%BA%E4%BE%8B%E6%9D%A5%E8%AF%B4%E6%98%8E%E5%9C%A8map%E9%87%8C%E5%A6%82%E4%BD%95%E6%A0%B9%E6%8D%AEkey%E6%9D%A5%E8%8E%B7%E5%8F%96value%E3%80%82%0A%60%60%60%0Afunc%20mapaccess1\(t%20\*maptype%2C%20h%20\*hmap%2C%20key%20unsafe.Pointer\)%20unsafe.Pointer%20%7B%0A%0A%09%2F%2F%E7%9C%81%E7%95%A5%E4%B8%80%E4%BA%9B%E5%89%8D%E7%BD%AE%E5%88%A4%E6%96%AD...%0A%0A%09%2F%2F%E5%A6%82%E6%9E%9C%E5%B9%B6%E5%8F%91%E8%AF%BB%E5%86%99%20%E6%8A%9B%E5%87%BA%E5%BC%82%E5%B8%B8%0A%09if%20h.flags%26hashWriting%20\!%3D%200%20%7B%0A%09%09throw\(%22concurrent%20map%20read%20and%20map%20write%22\)%0A%09%7D%0A%0A%09%2F%2F%E5%AF%B9key%E8%BF%9B%E8%A1%8Chash%0A%09hash%20%3A%3D%20t.hasher\(key%2C%20uintptr\(h.hash0\)\)%0A%0A%09%2F%2F%E4%BB%8Ehmap.buckets%E8%8E%B7%E5%8F%96key%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%E7%9A%84%20%0A%09m%20%3A%3D%20bucketMask\(h.B\)%0A%09b%20%3A%3D%20\(\*bmap\)\(add\(h.buckets%2C%20\(hash%26m\)\*uintptr\(t.bucketsize\)\)\)%0A%0A%0A%09%2F%2F%E6%98%AF%E5%90%A6%E6%AD%A3%E5%9C%A8%E6%89%A9%E5%AE%B9%E9%98%B6%E6%AE%B5%0A%09if%20c%20%3A%3D%20h.oldbuckets%3B%20c%20\!%3D%20nil%20%7B%0A%09%09if%20\!h.sameSizeGrow\(\)%20%7B%0A%09%09%09%2F%2F%20There%20used%20to%20be%20half%20as%20many%20buckets%3B%20mask%20down%20one%20more%20power%20of%20two.%0A%09%09%09m%20%3E%3E%3D%201%0A%09%09%7D%0A%09%09oldb%20%3A%3D%20\(\*bmap\)\(add\(c%2C%20\(hash%26m\)\*uintptr\(t.bucketsize\)\)\)%0A%09%09if%20\!evacuated\(oldb\)%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%2F%2F%E8%8B%A5%E6%AD%A3%E5%9C%A8%E6%89%A9%E5%AE%B9%EF%BC%8C%E5%88%99%E5%88%B0%E8%80%81%E7%9A%84%20buckets%20%E4%B8%AD%E6%9F%A5%E6%89%BE%EF%BC%88%E5%9B%A0%E4%B8%BA%20buckets%20%E4%B8%AD%E5%8F%AF%E8%83%BD%E8%BF%98%E6%B2%A1%E6%9C%89%E5%80%BC%EF%BC%8C%E6%90%AC%E8%BF%81%E6%9C%AA%E5%AE%8C%E6%88%90%EF%BC%89%0A%09%09%09b%20%3D%20oldb%0A%09%09%7D%0A%09%7D%0A%0A%09%2F%2F%E8%8E%B7%E5%8F%96hash%E5%80%BC%E7%9A%84%E9%AB%988%E4%BD%8D%0A%09top%20%3A%3D%20tophash\(hash\)%0Abucketloop%3A%0A%09for%20%3B%20b%20\!%3D%20nil%3B%20b%20%3D%20b.overflow\(t\)%20%7B%20%20%2F%2F%E5%85%88%E5%9C%A8%E6%AD%A3%E5%B8%B8%E6%A1%B6%E9%87%8C%E6%89%BE%20%E6%89%BE%E4%B8%8D%E5%88%B0%E5%B0%B1%E5%9C%A8%E6%BA%A2%E5%87%BA%E6%A1%B6%E9%87%8C%E6%89%BE%0A%09%09%2F%2F%E6%A0%B9%E6%8D%AE%E8%AE%A1%E7%AE%97%E5%87%BA%E6%9D%A5%E7%9A%84%E5%93%88%E5%B8%8C%E5%80%BC%E7%9A%84%E9%AB%988%E4%BD%8D%20%E4%BE%9D%E6%AC%A1%E4%B8%8E%E6%A1%B6%E9%87%8C%E5%B7%B2%E7%BB%8F%E5%AD%98%E5%82%A8%E7%9A%84tophash%E7%9A%84%E8%BF%9B%E8%A1%8C%E6%AF%94%E8%BE%83%EF%BC%88%E5%BF%AB%E9%80%9F%E8%AF%95%E9%94%99%EF%BC%8C%E4%B8%8D%E4%BC%9A%E4%B8%80%E5%BC%80%E5%A7%8B%E5%B0%B1%E7%9B%B4%E6%8E%A5%E6%8B%BFkey%E6%9D%A5%E8%BF%9B%E8%A1%8C%E5%AF%B9%E6%AF%94%EF%BC%89%0A%09%09for%20i%20%3A%3D%20uintptr\(0\)%3B%20i%20%3C%20bucketCnt%3B%20i%2B%2B%20%7B%0A%09%09%09if%20b.tophash%5Bi%5D%20\!%3D%20top%20%7B%0A%09%09%09%09if%20b.tophash%5Bi%5D%20%3D%3D%20emptyRest%20%7B%20%2F%2F%E8%BF%99%E4%B8%AA%E6%A1%B6%E5%90%8E%E9%9D%A2%E9%83%BD%E6%B2%A1%E6%9C%89kv%E4%BA%86%EF%BC%8C%E7%9B%B4%E6%8E%A5%E9%80%80%E5%87%BA%E5%BD%93%E5%89%8D%E6%A1%B6%E7%9A%84%E9%81%8D%E5%8E%86%0A%09%09%09%09%09break%20bucketloop%0A%09%09%09%09%7D%0A%09%09%09%09continue%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%8F%96%E5%87%BA%E6%A1%B6%E9%87%8C%E7%9A%84key%0A%09%09%09k%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2Bi\*uintptr\(t.keysize\)\)%0A%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09k%20%3D%20\*\(\(\*unsafe.Pointer\)\(k\)\)%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%88%A4%E6%96%AD%E6%A1%B6%E9%87%8C%E7%9A%84key%E4%B8%8E%E6%89%80%E9%9C%80%E8%A6%81%E6%89%BE%E5%88%B0key%E6%98%AF%E5%90%A6%E7%9B%B8%E7%AD%89%0A%09%09%09if%20t.key.equal\(key%2C%20k\)%20%7B%0A%09%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E7%9B%B8%E7%AD%89%20%E5%8F%96%E5%87%BAkey%E5%AF%B9%E5%BA%94%E7%9A%84value%20%E7%84%B6%E5%90%8E%E8%BF%94%E5%9B%9E%0A%09%09%09%09e%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2BbucketCnt\*uintptr\(t.keysize\)%2Bi\*uintptr\(t.elemsize\)\)%0A%09%09%09%09if%20t.indirectelem\(\)%20%7B%0A%09%09%09%09%09e%20%3D%20\*\(\(\*unsafe.Pointer\)\(e\)\)%0A%09%09%09%09%7D%0A%09%09%09%09return%20e%0A%09%09%09%7D%0A%09%09%7D%0A%09%7D%0A%0A%09%2F%2F%E6%89%BE%E4%B8%8D%E5%88%B0%20%E8%BF%94%E5%9B%9E%E7%B1%BB%E5%9E%8B%E9%9B%B6%E5%80%BC%0A%09return%20unsafe.Pointer\(%26zeroVal%5B0%5D\)%0A%7D%0A%60%60%60%0A%E5%9C%A8%E4%B8%8A%E9%9D%A2%E8%AE%A1%E7%AE%97key%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%E7%9A%84%E8%BF%87%E7%A8%8B%E4%B8%AD%EF%BC%8C%E4%B8%80%E8%88%AC%E6%9D%A5%E8%AF%B4%EF%BC%8C%E6%A0%B9%E6%8D%AEmap%E7%9A%84hash%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97%E5%87%BAkey%E5%AF%B9%E5%BA%94%E7%9A%84hash%E5%80%BC%E5%90%8E%EF%BC%8C%E9%9C%80%E8%A6%81%E7%94%A8%E8%BF%99%E4%B8%AAhash%E5%80%BC%E5%AF%B9hmap.buckets%E7%9A%84%E9%95%BF%E5%BA%A6%E5%8F%96%E6%A8%A1%EF%BC%88%E5%9B%A0%E4%B8%BA%E5%93%88%E5%B8%8C%E5%80%BC%E5%BE%88%E5%8F%AF%E8%83%BD%E5%A4%A7%E4%BA%8Ebucket%E6%95%B0%E7%BB%84%E7%9A%84%E9%95%BF%E5%BA%A6%EF%BC%8C%E6%89%80%E4%BB%A5%E9%9C%80%E8%A6%81%E5%8F%96%E6%A8%A1%E6%9D%A5%E7%A1%AE%E4%BF%9Dhash%E5%80%BC%E8%83%BD%E5%AF%B9%E8%90%BD%E5%88%B0%E5%BA%94%E5%88%B0buckets%E4%B8%8A%EF%BC%89%E3%80%82%E4%B8%8D%E8%BF%87%E8%BF%99%E9%87%8C%E5%B9%B6%E6%B2%A1%E6%9C%89%E9%87%87%E7%94%A8%20%3Cu%3Ehash%E5%80%BC%20%25%20bucketsize%3C%2Fu%3E%20%E6%96%B9%E6%B3%95%E8%AE%A1%E7%AE%97%E5%AF%B9%E5%BA%94%E7%9A%84bucket%EF%BC%8C%E8%80%8C%E6%98%AF%E4%BD%BF%E7%94%A8%20%3Cu%3Ehasn%E5%80%BC%20%26%20bucketsize\-1%3C%2Fu%3E%20%E7%9A%84%E6%96%B9%E6%B3%95%E6%9D%A5\*\*%E5%BF%AB%E9%80%9F%E8%AE%A1%E7%AE%97\*\*%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%E7%9A%84%E4%BD%8D%E7%BD%AE%EF%BC%88%E4%B9%9F%E8%AE%B8%E6%98%AF%E5%9B%A0%E4%B8%BA%E4%BD%8D%E6%93%8D%E4%BD%9C%E6%AF%94%E5%8F%96%E6%A8%A1%E6%93%8D%E4%BD%9C%E5%BF%AB%EF%BC%89%EF%BC%8C\*\*%E4%BD%86%E6%98%AF%E8%BF%99%E7%A7%8D%E4%BD%8D%E6%93%8D%E4%BD%9C%E4%BB%A3%E6%9B%BF%E5%8F%96%E6%A8%A1%E7%9A%84%E5%89%8D%E6%8F%90%E6%98%AF%EF%BC%8Cbucket%E6%95%B0%E7%BB%84%E9%95%BF%E5%BA%A6%E5%BF%85%E9%A1%BB%E6%98%AF2%E7%9A%84%E6%8C%87%E6%95%B0\*\*%EF%BC%8C%E8%BF%99%E9%87%8C%E4%B9%9F%E5%B0%B1%E8%A7%A3%E9%87%8A%E4%BA%86%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E7%9B%B4%E6%8E%A5%E7%94%A8%E4%B8%80%E4%B8%AAint%E5%8F%98%E9%87%8F%E6%9D%A5%E5%AD%98%E5%82%A8bucketsize%EF%BC%8C%E8%80%8C%E6%98%AF%E4%BD%BF%E7%94%A8hmap.B%EF%BC%8C%E5%9B%A0%E4%B8%BAbucketsize%E9%83%BD%E6%98%AF%202%20%5E%20B%EF%BC%8C\*\*%E6%89%80%E4%BB%A5%E5%8F%AF%E4%BB%A5%E7%94%A8%E4%BD%8D%E6%93%8D%E4%BD%9C%E6%9D%A5%E4%BB%A3%E6%9B%BF%E5%8F%96%E6%A8%A1%E6%93%8D%E4%BD%9C%E3%80%82\*\*%0A%0A%23%23%23%23%20map%E7%9A%84%E5%86%99%E5%85%A5%0A%0Amap%E7%9A%84%E5%86%99%E5%85%A5%E4%BD%BF%E7%94%A8%E7%9A%84%E6%98%AFruntime.mapassign%E5%87%BD%E6%95%B0%EF%BC%8C%E8%AF%A5%E5%87%BD%E6%95%B0%E4%B8%8Eruntime.mapaccess1%E6%AF%94%E8%BE%83%E7%B1%BB%E4%BC%BC%E3%80%82%0A%60%60%60%0Afunc%20mapassign\(t%20\*maptype%2C%20h%20\*hmap%2C%20key%20unsafe.Pointer\)%20unsafe.Pointer%20%7B%0A%09%0A%09%2F%2F%E7%9C%81%E7%95%A5%E4%B8%80%E4%BA%9B%E5%89%8D%E7%BD%AE%E5%88%A4%E6%96%AD...%0A%0A%09%2F%2F%E4%B8%8D%E5%85%81%E8%AE%B8%E5%B9%B6%E5%8F%91%E5%86%99%0A%09if%20h.flags%26hashWriting%20\!%3D%200%20%7B%0A%09%09throw\(%22concurrent%20map%20writes%22\)%0A%09%7D%0A%09%0A%09%2F%2F%E9%80%9A%E8%BF%87%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97key%E5%AF%B9%E5%BA%94%E7%9A%84%E5%93%88%E5%B8%8C%E5%80%BC%0A%09hash%20%3A%3D%20t.hasher\(key%2C%20uintptr\(h.hash0\)\)%0A%0A%09%2F%2F%E6%A0%87%E8%AE%B0%E4%B8%BA%E6%AD%A3%E5%9C%A8%E5%86%99%0A%09h.flags%20%5E%3D%20hashWriting%0A%0A%09%2F%2F%E5%A6%82%E6%9E%9Chmap.buckets%E6%B2%A1%E6%9C%89%E5%88%9D%E5%A7%8B%E5%8C%96%20%E5%88%99%E5%88%9D%E5%A7%8B%E5%8C%96buckets%0A%09if%20h.buckets%20%3D%3D%20nil%20%7B%0A%09%09h.buckets%20%3D%20newobject\(t.bucket\)%20%2F%2F%20newarray\(t.bucket%2C%201\)%0A%09%7D%0A%0Aagain%3A%0A%20%20%20%20%0A%09bucket%20%3A%3D%20hash%20%26%20bucketMask\(h.B\)%0A%20%20%20%20%0A%09%2F%2F%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8D%E6%AD%A3%E5%9C%A8%E6%89%A9%E5%AE%B9%E9%98%B6%E6%AE%B5%20%E9%82%A3%E4%B9%88%E5%85%88%E5%B0%86%E6%89%A9%E5%AE%B9%E5%AE%8C%E6%88%90%0A%09if%20h.growing\(\)%20%7B%0A%09%09growWork\(t%2C%20h%2C%20bucket\)%0A%09%7D%0A%0A%09%2F%2F%E6%89%BE%E5%88%B0%E6%A1%B6%E7%9A%84%E4%BD%8D%E7%BD%AE%0A%09b%20%3A%3D%20\(\*bmap\)\(add\(h.buckets%2C%20bucket\*uintptr\(t.bucketsize\)\)\)%0A%09top%20%3A%3D%20tophash\(hash\)%0A%0A%09var%20inserti%20\*uint8%0A%09var%20insertk%20unsafe.Pointer%0A%09var%20elem%20unsafe.Pointer%0Abucketloop%3A%0A%09for%20%7B%0A%09%09for%20i%20%3A%3D%20uintptr\(0\)%3B%20i%20%3C%20bucketCnt%3B%20i%2B%2B%20%7B%20%2F%2F%E9%81%8D%E5%8E%86%E6%A1%B6%E9%87%8C%E7%9A%848%E4%B8%AA%E6%A7%BD%0A%09%09%09if%20b.tophash%5Bi%5D%20\!%3D%20top%20%7B%0A%09%09%09%09%2F%2F%E8%8B%A5%E9%81%8D%E5%8E%86%E5%88%B0%E7%9A%84%E6%A7%BD%E7%9A%84tophash%E4%B8%BA%E7%A9%BA%2C%20%E5%88%99%E5%85%88%E8%AE%B0%E5%BD%95%E4%B8%8B%E8%AF%A5%E4%BD%8D%E7%BD%AE%20%20%E5%90%8E%E9%9D%A2%E5%8F%AF%E8%83%BD%E7%9B%B4%E6%8E%A5%E7%94%A8%E8%BF%99%E4%B8%AA%E4%BD%8D%E7%BD%AE%E6%9D%A5%E6%94%BE%E7%BD%AEkv%E9%94%AE%E5%80%BC%E5%AF%B9%0A%09%09%09%09if%20isEmpty\(b.tophash%5Bi%5D\)%20%26%26%20inserti%20%3D%3D%20nil%20%7B%20%0A%09%09%09%09%09inserti%20%3D%20%26b.tophash%5Bi%5D%0A%09%09%09%09%09insertk%20%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2Bi\*uintptr\(t.keysize\)\)%0A%09%09%09%09%09elem%20%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2BbucketCnt\*uintptr\(t.keysize\)%2Bi\*uintptr\(t.elemsize\)\)%0A%09%09%09%09%7D%0A%09%09%09%09%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%E8%AF%A5%E6%A7%BD%E5%90%8E%E9%9D%A2%E9%83%BD%E6%B2%A1%E6%9C%89kv%E4%BA%86%20%E6%B2%A1%E6%9C%89%E5%BF%85%E8%A6%81%E6%89%BE%E4%BA%86%0A%09%09%09%09if%20b.tophash%5Bi%5D%20%3D%3D%20emptyRest%20%7B%0A%09%09%09%09%09break%20bucketloop%0A%09%09%09%09%7D%0A%09%09%09%09continue%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E6%A7%BD%E7%9A%84tophash%E4%B8%8E%E5%8F%82%E6%95%B0key%E7%9A%84tophash%E7%9B%B8%E7%AD%89%20%E8%BF%9B%E4%B8%80%E6%AD%A5%E5%8F%96%E5%87%BA%E6%A7%BD%E5%AF%B9%E5%BA%94%E7%9A%84key%20%0A%09%09%09k%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2Bi\*uintptr\(t.keysize\)\)%0A%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09k%20%3D%20\*\(\(\*unsafe.Pointer\)\(k\)\)%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%88%A4%E6%96%AD%20%E6%A7%BD%E5%AF%B9%E5%BA%94%E7%9A%84key%20%E4%B8%8E%20%E5%85%A5%E5%8F%82key%20%E6%98%AF%E5%90%A6%E7%9B%B8%E7%AD%89%0A%09%09%09if%20\!t.key.equal\(key%2C%20k\)%20%7B%0A%09%09%09%09continue%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E7%9B%B8%E7%AD%89%20%E5%88%99%E8%AF%B4%E6%98%8Emap%E5%B7%B2%E7%BB%8F%E5%AD%98%E5%9C%A8%E8%BF%99%E4%B8%AAkey%E4%BA%86%20%E6%9B%B4%E6%96%B0key%E5%AF%B9%E5%BA%94%E7%9A%84value%0A%09%09%09if%20t.needkeyupdate\(\)%20%7B%0A%09%09%09%09typedmemmove\(t.key%2C%20k%2C%20key\)%0A%09%09%09%7D%0A%09%09%09elem%20%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2BbucketCnt\*uintptr\(t.keysize\)%2Bi\*uintptr\(t.elemsize\)\)%0A%0A%09%09%09%2F%2F%E6%9B%B4%E6%96%B0%E5%AE%8C%E6%88%90%20%E7%9B%B4%E6%8E%A5%E6%B8%85%E9%99%A4%E5%86%99%E6%A0%87%E5%BF%97%E4%BD%8D%E7%84%B6%E5%90%8E%E8%BF%94%E5%9B%9E%0A%09%09%09goto%20done%0A%09%09%7D%0A%0A%09%09%2F%2F%E6%AD%A3%E5%B8%B8%E6%A1%B6%E9%87%8C%E6%B2%A1%E6%9C%89%20%E7%BB%A7%E7%BB%AD%E5%9C%A8%E6%BA%A2%E5%87%BA%E6%A1%B6%E9%87%8C%E6%89%BE%0A%09%09ovf%20%3A%3D%20b.overflow\(t\)%0A%09%09if%20ovf%20%3D%3D%20nil%20%7B%0A%09%09%09break%0A%09%09%7D%0A%09%09b%20%3D%20ovf%0A%09%7D%0A%0A%0A%09%2F%2F%E5%88%B0%E8%BF%99%E9%87%8C%E8%AF%B4%E8%AF%B4%E6%98%8E%E5%BD%93%E5%89%8Dmap%E4%B8%8D%E5%AD%98%E5%9C%A8%E8%BF%99%E4%B8%AAkey%0A%09%2F%2F%E5%A6%82%E6%9E%9C%E5%BD%93%E5%89%8D%E6%B2%A1%E6%9C%89%E6%AD%A3%E5%9C%A8%E6%89%A9%E5%AE%B9%20%E5%86%8D%E5%88%A4%E6%96%AD%E4%B8%8B%E9%9D%A2%E4%B8%A4%E4%B8%AA%E6%9D%A1%E4%BB%B6%EF%BC%9A%0A%09%2F%2F1.%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%EF%BC%88%E5%85%83%E7%B4%A0%E6%95%B0%E9%87%8F%2F%E6%A1%B6%E7%9A%84%E6%95%B0%E9%87%8F%EF%BC%89%20%3E%206.5%0A%09%2F%2F2.map%E4%BD%BF%E7%94%A8%E5%A4%AA%E5%A4%9A%E6%BA%A2%E5%87%BA%E6%A1%B6%E4%BA%86%0A%09%2F%2F%E5%A6%82%E6%9E%9C%E6%BB%A1%E8%B6%B3%E5%85%B6%E4%B8%AD%E4%BB%BB%E6%84%8F%E4%B8%80%E4%B8%AA%E6%9D%A1%E4%BB%B6%20%E5%88%99%E8%BF%9B%E5%85%A5%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B%0A%0A%09if%20\!h.growing\(\)%20%26%26%20\(overLoadFactor\(h.count%2B1%2C%20h.B\)%20%7C%7C%20tooManyOverflowBuckets\(h.noverflow%2C%20h.B\)\)%20%7B%0A%09%09hashGrow\(t%2C%20h\)%20%2F%2F%E6%89%A9%E5%AE%B9%0A%09%09goto%20again%20%2F%2F%20%E9%87%8D%E6%96%B0%E5%88%A4%E6%96%AD%0A%09%7D%0A%0A%0A%09%2F%2F%E4%B8%8D%E9%9C%80%E8%A6%81%E6%89%A9%E5%AE%B9%20%E5%88%A4%E6%96%AD%E5%9C%A8%E5%BD%93%E5%89%8D%E6%A1%B6%EF%BC%88%E6%AD%A3%E5%B8%B8%E6%A1%B6%EF%BC%89%E9%87%8C%E6%98%AF%E5%90%A6%E8%83%BD%E6%9C%89%E6%94%BE%E6%96%B0%E7%9A%84kv%E7%9A%84%E4%BD%8D%E7%BD%AE%20%0A%09if%20inserti%20%3D%3D%20nil%20%7B%0A%09%09%2F%2F%E6%AD%A3%E5%B8%B8%E6%A1%B6%E9%87%8C%E6%B2%A1%E6%9C%89%E4%BD%8D%E7%BD%AE%E4%BA%86%20%E6%96%B0%E7%94%9F%E6%88%90%E4%B8%80%E4%B8%AA%E6%BA%A2%E5%87%BA%E6%A1%B6%E6%9D%A5%E5%AD%98%E6%94%BE%20%E5%B0%86%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E7%9A%84%E8%AE%BE%E7%BD%AE%E4%B8%BAkv%E6%8F%92%E5%85%A5%E7%9A%84%E4%BD%8D%E7%BD%AE%0A%09%09newb%20%3A%3D%20h.newoverflow\(t%2C%20b\)%0A%09%09inserti%20%3D%20%26newb.tophash%5B0%5D%0A%09%09insertk%20%3D%20add\(unsafe.Pointer\(newb\)%2C%20dataOffset\)%0A%09%09elem%20%3D%20add\(insertk%2C%20bucketCnt\*uintptr\(t.keysize\)\)%0A%09%7D%0A%0A%09%2F%2F%20%E5%B0%86kv%E6%94%BE%E5%85%A5%E5%AF%B9%E5%BA%94%E6%8F%92%E5%85%A5%E4%BD%8D%E7%BD%AE%0A%09if%20t.indirectkey\(\)%20%7B%0A%09%09kmem%20%3A%3D%20newobject\(t.key\)%0A%09%09\*\(\*unsafe.Pointer\)\(insertk\)%20%3D%20kmem%0A%09%09insertk%20%3D%20kmem%0A%09%7D%0A%09if%20t.indirectelem\(\)%20%7B%0A%09%09vmem%20%3A%3D%20newobject\(t.elem\)%0A%09%09\*\(\*unsafe.Pointer\)\(elem\)%20%3D%20vmem%0A%09%7D%0A%09typedmemmove\(t.key%2C%20insertk%2C%20key\)%0A%09\*inserti%20%3D%20top%0A%09h.count%2B%2B%0A%0A%0A%09%2F%2F%E6%B8%85%E9%99%A4%E5%86%99%E6%A0%87%E5%BF%97%E4%BD%8D%0Adone%3A%0A%09if%20h.flags%26hashWriting%20%3D%3D%200%20%7B%0A%09%09throw\(%22concurrent%20map%20writes%22\)%0A%09%7D%0A%09h.flags%20%26%5E%3D%20hashWriting%0A%09if%20t.indirectelem\(\)%20%7B%0A%09%09elem%20%3D%20\*\(\(\*unsafe.Pointer\)\(elem\)\)%0A%09%7D%0A%09return%20elem%0A%7D%0A%0A%60%60%60%0A%0Amap%E6%89%A9%E5%AE%B9%E7%9A%84%E6%9D%A1%E4%BB%B6%E6%9C%89%E4%B8%A4%E4%B8%AA%EF%BC%8C%E5%88%86%E5%88%AB%E5%AF%B9%E5%BA%94%E4%B8%A4%E7%A7%8D%E6%89%A9%E5%AE%B9%E6%96%B9%E5%BC%8F%E3%80%82%0A%0A1.%20\*\*%E5%A6%82%E6%9E%9C%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%3E6.5\*\*%20map%E7%9A%84%E6%89%A9%E5%AE%B9%E7%AD%96%E7%95%A5%E4%B8%BA%E5%B0%86hmap.B%E5%8A%A0%E4%B8%80%2C%20%E5%8D%B3%E5%B0%86%E6%95%B4%E4%B8%AA%E5%93%88%E5%B8%8C%E6%A1%B6%E6%95%B0%E7%9B%AE%E6%89%A9%E5%85%85%E4%B8%BA%E5%8E%9F%E6%9D%A5%E7%9A%84%E4%B8%A4%E5%80%8D%E5%A4%A7%E5%B0%8F%2C%20%E8%BF%99%E7%A7%8D%E7%AD%96%E7%95%A5%E5%B1%9E%E4%BA%8E\*\*%E5%A2%9E%E9%87%8F%E6%89%A9%E5%AE%B9\*\*%E3%80%82%0A2.%20\*\*%E5%A6%82%E6%9E%9C%E6%98%AF%E6%BA%A2%E5%87%BA%E6%A1%B6%E8%BF%87%E5%A4%9A\*\*%EF%BC%8C%E4%B9%9F%E4%BC%9A%E8%A7%A6%E5%8F%91%E6%89%A9%E5%AE%B9%E3%80%82%E4%B8%BA%E4%BB%80%E4%B9%88%E5%A4%AA%E5%A4%9A%E6%BA%A2%E5%87%BA%E6%A1%B6%E5%A4%AA%EF%BC%8C%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%E5%B9%B6%E6%B2%A1%E6%9C%89%E8%B6%85%E8%BF%87%E9%98%88%E5%80%BC%EF%BC%8C%E4%B9%9F%E4%BC%9A%E9%9C%80%E8%A6%81%E6%89%A9%E5%AE%B9%EF%BC%9F%E8%80%83%E8%99%91%E8%BF%99%E4%B9%88%E4%B8%80%E4%B8%AAcase%2C%20%E5%90%91%20map%20%E4%B8%AD%E6%8F%92%E5%85%A5%E5%A4%A7%E9%87%8F%E7%9A%84%E5%85%83%E7%B4%A0%2C%20%E5%93%88%E5%B8%8C%E6%A1%B6%E5%B0%86%E9%80%90%E6%B8%90%E8%A2%AB%E5%A1%AB%E6%BB%A1%2C%20%E8%BF%99%E4%B8%AA%E8%BF%87%E7%A8%8B%E4%B8%AD%E4%B9%9F%E5%8F%AF%E8%83%BD%E5%88%9B%E5%BB%BA%E4%BA%86%E4%B8%80%E4%BA%9B%E6%BA%A2%E5%87%BA%E6%A1%B6%2C%20%E4%BD%86%E6%AD%A4%E6%97%B6%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%E5%B9%B6%E6%B2%A1%E6%9C%89%E8%B6%85%E8%BF%87%E8%AE%BE%E5%AE%9A%E7%9A%84%E9%98%88%E5%80%BC%2C%20%E7%84%B6%E5%90%8E%E5%AF%B9%E8%BF%99%E4%BA%9B%20map%20%E5%81%9A%E5%88%A0%E9%99%A4%E6%93%8D%E4%BD%9C%2C%20%E5%88%A0%E9%99%A4%E5%85%83%E7%B4%A0%E4%B9%8B%E5%90%8E%2C%20map%20%E4%B8%AD%E7%9A%84%E5%85%83%E7%B4%A0%E6%95%B0%E7%9B%AE%E5%8F%98%E5%B0%91%2C%20%E4%BD%BF%E5%BE%97%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%E9%99%8D%E4%BD%8E%2C%20%E8%80%8C%E5%90%8E%E5%8F%88%E9%87%8D%E5%A4%8D%E4%B8%8A%E8%BF%B0%E7%9A%84%E8%BF%87%E7%A8%8B%2C%20%E6%9C%80%E7%BB%88%E4%BD%BF%E5%BE%97%E6%95%B4%E4%BD%93%E7%9A%84%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%E4%B8%8D%E5%A4%A7%2C%20%E4%BD%86%E6%95%B4%E4%B8%AA%20map%20%E4%B8%AD%E5%AD%98%E5%9C%A8%E4%BA%86%E5%A4%A7%E9%87%8F%E7%9A%84%E6%BA%A2%E5%87%BA%E6%A1%B6%2C%20%E5%9B%A0%E6%AD%A4%E5%BD%93%E6%BA%A2%E5%87%BA%E6%A1%B6%E6%95%B0%E7%9B%AE%E8%BF%87%E5%A4%9A%E6%97%B6%2C%20%E5%8D%B3%E4%BE%BF%E6%B2%A1%E6%9C%89%E8%BE%BE%E5%88%B0%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%206.5%20%E7%9A%84%E9%98%88%E5%80%BC%E4%B9%9F%E4%BC%9A%E8%A7%A6%E5%8F%91%E6%89%A9%E5%AE%B9%E3%80%82%E8%BF%99%E7%A7%8D%E6%89%A9%E5%AE%B9%E5%B1%9E%E4%BA%8E\*\*%E7%AD%89%E9%87%8F%E6%89%A9%E5%AE%B9\*\*%20%E3%80%82%0A%0A%E5%BD%93%E6%BB%A1%E8%B6%B3%E6%89%A9%E5%AE%B9%E6%9D%A1%E4%BB%B6%E6%97%B6%EF%BC%8C%E5%B0%B1%E4%BC%9A%E8%B0%83%E7%94%A8runtime.hashGrow%EF%BC%8C%E4%BD%86runtime.hashGrow%E5%8F%AA%E4%BC%9A%E6%9E%84%E9%80%A0%E6%96%B0%E6%A1%B6%E5%B9%B6%E4%B8%94%E4%BF%AE%E6%94%B9%E7%9B%B8%E5%85%B3%E5%AD%97%E6%AE%B5%EF%BC%8C\*\*%E5%B9%B6%E4%B8%8D%E4%BC%9A%E7%9C%9F%E6%AD%A3%E6%8A%8A%E6%95%B0%E6%8D%AE%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E6%A1%B6%E4%B8%8A\*\*%E3%80%82\*\*%E5%8F%AA%E6%9C%89%E5%BD%93map%E8%BF%9B%E8%A1%8C%E6%8F%92%E5%85%A5%E8%B7%9F%E5%88%A0%E9%99%A4%E6%93%8D%E4%BD%9C%EF%BC%8C%E6%89%8D%E4%BC%9A%E6%8A%8A%E6%95%B0%E6%8D%AE%E4%BB%8E%E6%97%A7%E6%A1%B6%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E6%A1%B6%EF%BC%8C%E8%BF%81%E7%A7%BB%E7%9A%84%E8%BF%87%E7%A8%8B%E6%98%AFruntime.growWork%E5%AE%8C%E6%88%90%E7%9A%84\*\*%E3%80%82%0A%60%60%60%0Afunc%20hashGrow\(t%20\*maptype%2C%20h%20\*hmap\)%20%7B%0A%09%0A%09bigger%20%3A%3D%20uint8\(1\)%0A%09if%20\!overLoadFactor\(h.count%2B1%2C%20h.B\)%20%7B%0A%09%09bigger%20%3D%200%0A%09%09h.flags%20%7C%3D%20sameSizeGrow%20%2F%2F%E5%A6%82%E6%9E%9C%E8%A3%85%E8%BD%BD%E5%9B%A0%E5%AD%90%E4%B8%8D%E5%A4%A7%E4%BA%8E6.5%20%E5%88%99%E4%B8%BA%E5%A2%9E%E9%87%8F%E6%89%A9%E5%AE%B9%0A%09%7D%0A%0A%09%2F%2F%E4%BF%9D%E5%AD%98%E6%97%A7%E6%A1%B6%20%E5%B9%B6%E6%9E%84%E9%80%A0%E6%96%B0%E6%A1%B6%0A%09oldbuckets%20%3A%3D%20h.buckets%0A%09newbuckets%2C%20nextOverflow%20%3A%3D%20makeBucketArray\(t%2C%20h.B%2Bbigger%2C%20nil\)%0A%0A%09flags%20%3A%3D%20h.flags%20%26%5E%20\(iterator%20%7C%20oldIterator\)%0A%09if%20h.flags%26iterator%20\!%3D%200%20%7B%0A%09%09flags%20%7C%3D%20oldIterator%0A%09%7D%0A%09%0A%09%2F%2F%20%E7%9B%B8%E5%85%B3%E5%8F%82%E6%95%B0%E4%BF%AE%E6%94%B9%0A%09h.B%20%2B%3D%20bigger%0A%09h.flags%20%3D%20flags%0A%09h.oldbuckets%20%3D%20oldbuckets%0A%09h.buckets%20%3D%20newbuckets%0A%09h.nevacuate%20%3D%200%0A%09h.noverflow%20%3D%200%0A%0A%09%2F%2F%E5%B0%86%E5%BD%93%E5%89%8D%E6%BA%A2%E5%87%BA%E6%A1%B6%E8%B5%8B%E5%80%BC%E7%BB%99%E6%97%A7%E6%BA%A2%E5%87%BA%E6%A1%B6%20%E5%BD%93%E5%89%8D%E6%BA%A2%E5%87%BA%E6%A1%B6%E6%9B%B4%E6%96%B0%E4%B8%BAnil%0A%09if%20h.extra%20\!%3D%20nil%20%26%26%20h.extra.overflow%20\!%3D%20nil%20%7B%0A%09%09if%20h.extra.oldoverflow%20\!%3D%20nil%20%7B%0A%09%09%09throw\(%22oldoverflow%20is%20not%20nil%22\)%0A%09%09%7D%0A%09%09h.extra.oldoverflow%20%3D%20h.extra.overflow%0A%09%09h.extra.overflow%20%3D%20nil%0A%09%7D%0A%0A%09%2F%2F%E9%87%8D%E6%96%B0%E8%B5%8B%E5%80%BCnextOverflow%0A%09if%20nextOverflow%20\!%3D%20nil%20%7B%0A%09%09if%20h.extra%20%3D%3D%20nil%20%7B%0A%09%09%09h.extra%20%3D%20new\(mapextra\)%0A%09%09%7D%0A%09%09h.extra.nextOverflow%20%3D%20nextOverflow%0A%09%7D%0A%0A%7D%0A%60%60%60%0A%E4%B8%8B%E9%9D%A2%E6%98%AF%E8%A7%A6%E5%8F%91%E6%89%A9%E5%AE%B9%E5%90%8E%E7%9A%84%E5%93%88%E5%B8%8C%E3%80%82%0A\!%5B3a7e3be5ecc4fa77b61ac6166ca23ab3.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp109\)%0A%0A%0A\*\*runtime.growWork\*\*%E8%B4%9F%E8%B4%A3%E5%87%BA%E5%A4%84%E7%90%86%E6%A1%B6%E8%BF%81%E7%A7%BB%EF%BC%8C%E8%80%8C%E5%85%B6%E4%B8%AD%E5%8F%88%E8%B0%83%E7%94%A8\*\*runtime.evacuate\*\*%E6%9D%A5%E5%85%B7%E4%BD%93%E8%BF%81%E7%A7%BB%E6%9F%90%E4%B8%AA%E6%A1%B6%EF%BC%8C%E5%8F%AF%E4%BB%A5%E7%9C%8B%E5%88%B0%E4%B8%80%E6%AC%A1%E8%BF%81%E7%A7%BB%E8%BF%87%E7%A8%8B%E6%9C%80%E5%A4%9A%E6%90%AC%E7%A7%BB\*\*%E4%B8%A4%E4%B8%AA%E6%A1%B6\*\*%EF%BC%8C%E8%80%8C%E4%B8%8D%E6%98%AF%E4%B8%80%E6%AC%A1%E6%80%A7%E5%B0%86%E6%89%80%E6%9C%89%E6%97%A7%E6%A1%B6%E9%83%BD%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E6%A1%B6%EF%BC%8C%E8%BF%99%E6%A0%B7%E5%81%9A%E6%98%AF%E4%B8%BA%E4%BA%86\*\*%E9%81%BF%E5%85%8D%E4%B8%80%E6%AC%A1%E8%BF%81%E7%A7%BB%E5%A4%AA%E6%A1%B6%E5%A4%9A%E4%BC%9A%E9%80%A0%E6%88%90%E5%8C%BA%E9%97%B4%E6%80%A7%E8%83%BD%E6%8A%96%E5%8A%A8\*\*%E3%80%82%0A%60%60%60%0Afunc%20growWork\(t%20\*maptype%2C%20h%20\*hmap%2C%20bucket%20uintptr\)%20%7B%0A%09%2F%2F%20%E5%BD%93%E5%89%8D%E6%93%8D%E4%BD%9Ckey%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%EF%BC%88%E6%97%A7%E6%A1%B6%EF%BC%89%EF%BC%8C%E6%90%AC%E8%BF%81%E5%88%B0%E6%96%B0%E6%A1%B6%E6%95%B0%E7%BB%84%0A%09evacuate\(t%2C%20h%2C%20bucket%26h.oldbucketmask\(\)\)%0A%0A%09%2F%2F%20evacuate%20one%20more%20oldbucket%20to%20make%20progress%20on%20growing%0A%09if%20h.growing\(\)%20%7B%0A%09%09evacuate\(t%2C%20h%2C%20h.nevacuate\)%0A%09%7D%0A%7D%0A%60%60%60%0A%0A\*\*runtime.evacuate\*\*%E8%BF%81%E7%A7%BB%E6%B5%81%E7%A8%8B%E5%A6%82%E4%B8%8B%E3%80%82%0A%60%60%60%0Afunc%20evacuate\(t%20\*maptype%2C%20h%20\*hmap%2C%20oldbucket%20uintptr\)%20%7B%0A%09%2F%2F%E6%89%BE%E5%88%B0key%E5%AF%B9%E5%BA%94%E7%9A%84%E6%97%A7%E6%A1%B6%0A%09b%20%3A%3D%20\(\*bmap\)\(add\(h.oldbuckets%2C%20oldbucket\*uintptr\(t.bucketsize\)\)\)%0A%09%2F%2F%20newbit%E5%8D%B3%E4%B8%BA%E8%80%81%E6%A1%B6%E7%9A%84%E4%B8%AA%E6%95%B0%0A%09newbit%20%3A%3D%20h.noldbuckets\(\)%0A%0A%09if%20\!evacuated\(b\)%20%7B%0A%09%09%2F%2F%20xy%20%E6%98%AF%E6%8A%8A%E8%80%81%E6%A1%B6%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E6%A1%B6%E5%8E%BB%E7%9A%84%E4%BD%8D%E7%BD%AE%EF%BC%8Cx%E6%98%AF%E4%BD%8E%E4%BD%8D%EF%BC%8Cy%E6%98%AF%E9%AB%98%E4%BD%8D%0A%09%09var%20xy%20%5B2%5DevacDst%0A%09%09x%20%3A%3D%20%26xy%5B0%5D%0A%09%09%2F%2F%20x%E5%92%8C%E8%80%81%E6%A1%B6%E7%9A%84%E4%BD%8D%E7%BD%AE%E7%9B%B8%E5%90%8C%EF%BC%88%E4%BE%8B%E5%A6%82%E5%9C%A8%E8%80%81%E6%A1%B6%E6%95%B0%E7%BB%84%E4%B8%AD%E4%BD%8D%E7%BD%AE%E6%95%B0%E6%98%AF3%20%E9%82%A3%E4%B9%88%E8%BF%81%E7%A7%BB%E5%90%8E%E5%9C%A8%E6%96%B0%E6%A1%B6%E6%95%B0%E7%BB%84%E4%B8%AD%E4%B9%9F%E6%98%AF3%EF%BC%89%0A%09%09x.b%20%3D%20\(\*bmap\)\(add\(h.buckets%2C%20oldbucket\*uintptr\(t.bucketsize\)\)\)%0A%09%09%2F%2F%20%E6%96%B0%E6%A1%B6%E5%86%85k%E7%9A%84%E8%B5%B7%E5%A7%8B%E4%BD%8D%E7%BD%AE%0A%09%09x.k%20%3D%20add\(unsafe.Pointer\(x.b\)%2C%20dataOffset\)%0A%09%09%2F%2F%20%E6%96%B0%E6%A1%B6%E5%86%85v%E7%9A%84%E8%B5%B7%E5%A7%8B%E4%BD%8D%E7%BD%AE%0A%09%09x.e%20%3D%20add\(x.k%2C%20bucketCnt\*uintptr\(t.keysize\)\)%0A%0A%09%09if%20\!h.sameSizeGrow\(\)%20%7B%0A%09%09%09%2F%2F%E5%A2%9E%E9%87%8F%E6%89%A9%E5%AE%B9%20%E8%AE%BE%E7%BD%AEy%20x%E4%B8%8Ey%E4%B9%8B%E9%97%B4%E9%9A%94%E4%BA%86oldbuckets%E7%9A%84%E5%9C%B0%E5%9D%80%EF%BC%88%E4%BE%8B%E5%A6%82%E6%9C%894%E4%B8%AA%E6%A1%B6%EF%BC%8C%E6%9C%AC%E6%AC%A1%E8%BF%81%E7%A7%BB%E7%9A%84%E6%98%AF%E7%AC%AC%E4%B8%89%E4%B8%AA%E8%80%81%E6%A1%B6%EF%BC%8C%E9%82%A3%E4%B9%88x%E4%B8%BA3%EF%BC%8Cy%E4%B8%BA3%E5%8A%A0%E4%B8%8A%E8%80%81%E6%A1%B6%E4%B8%AA%E6%95%B04%EF%BC%8C%E5%8D%B3%E4%B8%BA7%EF%BC%89%0A%09%09%09y%20%3A%3D%20%26xy%5B1%5D%0A%09%09%09y.b%20%3D%20\(\*bmap\)\(add\(h.buckets%2C%20\(oldbucket%2Bnewbit\)\*uintptr\(t.bucketsize\)\)\)%0A%09%09%09y.k%20%3D%20add\(unsafe.Pointer\(y.b\)%2C%20dataOffset\)%0A%09%09%09y.e%20%3D%20add\(y.k%2C%20bucketCnt\*uintptr\(t.keysize\)\)%0A%09%09%7D%0A%0A%09%09for%20%3B%20b%20\!%3D%20nil%3B%20b%20%3D%20b.overflow\(t\)%20%7B%20%2F%2F%E9%81%8D%E5%8E%86%E8%80%81%E6%A1%B6%E5%8F%8A%E5%85%B6%E6%BA%A2%E5%87%BA%E6%A1%B6%0A%09%09%09%2F%2F%20%E8%80%81%E6%A1%B6%E5%86%85k%2Fv%E7%9A%84%E8%B5%B7%E5%A7%8B%E4%BD%8D%E7%BD%AE%0A%09%09%09k%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset\)%0A%09%09%09e%20%3A%3D%20add\(k%2C%20bucketCnt\*uintptr\(t.keysize\)\)%0A%09%09%09for%20i%20%3A%3D%200%3B%20i%20%3C%20bucketCnt%3B%20i%2C%20k%2C%20e%20%3D%20i%2B1%2C%20add\(k%2C%20uintptr\(t.keysize\)\)%2C%20add\(e%2C%20uintptr\(t.elemsize\)\)%20%7B%20%2F%2F%E9%81%8D%E5%8E%86%E8%80%81%E6%A1%B6%E5%86%85%E7%9A%84%E6%A7%BD%0A%09%09%09%09top%20%3A%3D%20b.tophash%5Bi%5D%0A%09%09%09%09if%20isEmpty\(top\)%20%7B%0A%09%09%09%09%09b.tophash%5Bi%5D%20%3D%20evacuatedEmpty%20%2F%2F%E6%A7%BD%E4%BD%8D%E7%BD%AE%E4%B8%BA%E7%A9%BA%EF%BC%8C%E6%A0%87%E8%AE%B0%E4%B8%BA%E6%A7%BD%E5%B7%B2%E8%BF%81%E7%A7%BB%0A%09%09%09%09%09continue%0A%09%09%09%09%7D%0A%09%09%09%09if%20top%20%3C%20minTopHash%20%7B%0A%09%09%09%09%09throw\(%22bad%20map%20state%22\)%0A%09%09%09%09%7D%0A%09%09%09%09k2%20%3A%3D%20k%0A%09%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09%09k2%20%3D%20\*\(\(\*unsafe.Pointer\)\(k2\)\)%0A%09%09%09%09%7D%0A%0A%09%09%09%09%0A%09%09%09%09var%20useY%20uint8%0A%09%09%09%09if%20\!h.sameSizeGrow\(\)%20%7B%0A%09%09%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E6%98%AF%E5%A2%9E%E9%87%8F%E6%89%A9%E5%AE%B9%20%E9%9C%80%E8%A6%81%E5%88%A4%E6%96%ADkey%E6%98%AF%E8%A6%81%E6%94%BE%E5%9C%A8x%E8%BF%98%E6%98%AF%E6%94%BE%E5%9C%A8y%0A%09%09%09%09%09hash%20%3A%3D%20t.hasher\(k2%2C%20uintptr\(h.hash0\)\)%0A%09%09%09%09%09if%20h.flags%26iterator%20\!%3D%200%20%26%26%20\!t.reflexivekey\(\)%20%26%26%20\!t.key.equal\(k2%2C%20k2\)%20%7B%0A%09%09%09%09%09%09%2F%2F%20%E5%A6%82%E6%9E%9C%E6%BB%A1%E8%B6%B3%E8%BF%99%E4%B8%80%E6%9D%A1%E4%BB%B6%EF%BC%88%E5%85%B7%E4%BD%93%E5%95%A5%E6%9D%A1%E4%BB%B6%E4%B8%8D%E6%98%AF%E5%BE%88%E8%83%BD%E7%9C%8B%E6%98%8E%E7%99%BD%E3%80%82%E3%80%82%E3%80%82%EF%BC%89%20%E7%9B%B4%E6%8E%A5%E8%BF%81%E7%A7%BB%E5%88%B0y%0A%09%09%09%09%09%09useY%20%3D%20top%20%26%201%0A%09%09%09%09%09%09top%20%3D%20tophash\(hash\)%0A%09%09%09%09%09%7D%20else%20%7B%0A%09%09%09%09%09%09%2F%2F%20%E6%A0%B9%E6%8D%AEhash%26newbit%20%E6%9D%A5%E5%88%A4%E6%96%AD%E5%BD%93%E5%89%8Dkey%E6%98%AF%E6%94%BE%E5%9C%A8x%E8%BF%98%E6%98%AFy%0A%09%09%09%09%09%09if%20hash%26newbit%20\!%3D%200%20%7B%0A%09%09%09%09%09%09%09useY%20%3D%201%0A%09%09%09%09%09%09%7D%0A%09%09%09%09%09%7D%0A%09%09%09%09%7D%0A%0A%09%09%09%09if%20evacuatedX%2B1%20\!%3D%20evacuatedY%20%7C%7C%20evacuatedX%5E1%20\!%3D%20evacuatedY%20%7B%0A%09%09%09%09%09throw\(%22bad%20evacuatedN%22\)%0A%09%09%09%09%7D%0A%0A%09%09%09%09%2F%2F%20%E6%A0%87%E8%AE%B0%E8%BF%99%E4%B8%AA%E6%A7%BD%E6%98%AF%E8%A2%AB%E8%BF%81%E7%A7%BB%E5%88%B0x%E8%BF%98%E6%98%AFy%0A%09%09%09%09b.tophash%5Bi%5D%20%3D%20evacuatedX%20%2B%20useY%20%2F%2F%20evacuatedX%20%2B%201%20%3D%3D%20evacuatedY%0A%09%09%09%09dst%20%3A%3D%20%26xy%5BuseY%5D%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F%20evacuation%20destination%0A%20%0A%20%09%09%09%09%2F%2F%20%E6%96%B0%E6%A1%B6%E7%9A%84%E6%A7%BD%E4%BD%8D%E6%BB%A1%E4%BA%86%EF%BC%8C%E9%9C%80%E8%A6%81%E7%BB%99%E6%96%B0%E6%A1%B6%E5%88%9B%E5%BB%BA%E6%BA%A2%E5%87%BA%E6%A1%B6%0A%09%09%09%09if%20dst.i%20%3D%3D%20bucketCnt%20%7B%0A%09%09%09%09%09%2F%2F%20%E8%BF%81%E7%A7%BB%E7%9B%AE%E7%9A%84%E5%9C%B0%E4%BF%AE%E6%94%B9%E4%B8%BA%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E4%BF%A1%E6%81%AF%0A%09%09%09%09%09dst.b%20%3D%20h.newoverflow\(t%2C%20dst.b\)%0A%09%09%09%09%09dst.i%20%3D%200%0A%09%09%09%09%09dst.k%20%3D%20add\(unsafe.Pointer\(dst.b\)%2C%20dataOffset\)%0A%09%09%09%09%09dst.e%20%3D%20add\(dst.k%2C%20bucketCnt\*uintptr\(t.keysize\)\)%0A%09%09%09%09%7D%0A%0A%09%09%09%09%2F%2F%E8%AE%BE%E7%BD%AE%E6%96%B0%E6%A1%B6%E6%A7%BD%E4%BD%8D%E7%9A%84tophash%0A%09%09%09%09dst.b.tophash%5Bdst.i%26\(bucketCnt\-1\)%5D%20%3D%20top%20%2F%2F%20mask%20dst.i%20as%20an%20optimization%2C%20to%20avoid%20a%20bounds%20check%0A%0A%09%09%09%09%2F%2F%E6%8A%8A%E8%80%81%E6%A1%B6%E7%9A%84kv%E5%A4%8D%E5%88%B6%E5%88%B0%E6%96%B0%E6%A1%B6%0A%09%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09%09\*\(\*unsafe.Pointer\)\(dst.k\)%20%3D%20k2%20%2F%2F%20copy%20pointer%0A%09%09%09%09%7D%20else%20%7B%0A%09%09%09%09%09typedmemmove\(t.key%2C%20dst.k%2C%20k\)%20%2F%2F%20copy%20elem%0A%09%09%09%09%7D%0A%09%09%09%09if%20t.indirectelem\(\)%20%7B%0A%09%09%09%09%09\*\(\*unsafe.Pointer\)\(dst.e\)%20%3D%20\*\(\*unsafe.Pointer\)\(e\)%0A%09%09%09%09%7D%20else%20%7B%0A%09%09%09%09%09typedmemmove\(t.elem%2C%20dst.e%2C%20e\)%0A%09%09%09%09%7D%0A%09%09%09%09%2F%2F%E6%96%B0%E6%A1%B6%E5%86%85%E8%BF%81%E7%A7%BB%E7%9A%84%E6%95%B0%E5%8A%A0%E4%B8%80%0A%09%09%09%09dst.i%2B%2B%0A%0A%09%09%09%09%2F%2F%E6%96%B0%E6%A1%B6%E7%9A%84kv%E5%90%91%E5%89%8D%E5%81%8F%E7%A7%BB%E4%B8%80%E4%B8%AA%E5%8D%95%E4%BD%8D%0A%09%09%09%09dst.k%20%3D%20add\(dst.k%2C%20uintptr\(t.keysize\)\)%0A%09%09%09%09dst.e%20%3D%20add\(dst.e%2C%20uintptr\(t.elemsize\)\)%0A%09%09%09%7D%0A%09%09%7D%0A%09%09%0A%09%09%2F%2F%20%E6%8A%8A%E8%80%81%E6%A1%B6%E9%87%8A%E6%94%BE%E6%8E%89%20%20%0A%09%09if%20h.flags%26oldIterator%20%3D%3D%200%20%26%26%20t.bucket.ptrdata%20\!%3D%200%20%7B%0A%09%09%09b%20%3A%3D%20add\(h.oldbuckets%2C%20oldbucket\*uintptr\(t.bucketsize\)\)%0A%0A%09%09%09%2F%2F%20%E5%8F%AA%E6%B8%85%E7%90%86kv%E7%9A%84%E5%86%85%E5%AD%98%EF%BC%8C%E8%80%8C%E4%B8%8D%E6%B8%85%E7%90%868%E4%B8%AA%E6%A7%BD%E7%9A%84tophash%EF%BC%8C%E8%BF%99%E9%87%8C%E7%95%99%E7%9D%80tophash%E4%B8%BB%E8%A6%81%E6%98%AF%E5%88%A9%E7%94%A8%E5%95%8A%E4%BB%96%E4%BB%AC%E7%9A%84%E6%A0%87%E8%AE%B0%E4%BD%8D%EF%BC%8C%E5%B7%B2%E6%96%B9%E4%BE%BF%E5%90%8E%E7%BB%AD%E7%9A%84%E5%85%B6%E4%BB%96%E7%9A%84%E6%A1%B6%E8%BF%81%E7%A7%BB%0A%09%09%09ptr%20%3A%3D%20add\(b%2C%20dataOffset\)%0A%09%09%09n%20%3A%3D%20uintptr\(t.bucketsize\)%20\-%20dataOffset%0A%09%09%09memclrHasPointers\(ptr%2C%20n\)%0A%09%09%7D%0A%09%7D%0A%0A%09%2F%2F%E8%BF%81%E7%A7%BB%E6%A0%87%E8%AE%B0%0A%09if%20oldbucket%20%3D%3D%20h.nevacuate%20%7B%0A%09%09advanceEvacuationMark\(h%2C%20t%2C%20newbit\)%0A%09%7D%0A%7D%0A%60%60%60%0A%E8%BF%81%E7%A7%BB%E4%BD%8D%E7%BD%AE%E7%A4%BA%E6%84%8F%E5%9B%BE%E3%80%82%0A\!%5B6c9038a0a055fe082a830d7d54cc3302.png%5D\(evernotecid%3A%2F%2F9423462E\-1591\-4A2D\-AF79\-F4FBBF0F308D%2Fappyinxiangcom%2F29941161%2FENResource%2Fp108\)%0A%0A%0A%0A%0A%0A%23%23%23%23%20map%E7%9A%84%E5%88%A0%E9%99%A4%0A%E4%BB%8Emap%E4%B8%AD%E5%88%A0%E9%99%A4%E5%85%83%E7%B4%A0%E4%BD%BF%E7%94%A8%E7%9A%84%E6%98%AFruntime.mapdelete%E5%87%BD%E6%95%B0%E7%B0%87%E4%B8%AD%E7%9A%84%E4%B8%80%E4%B8%AA%EF%BC%8C%E5%8C%85%E6%8B%AC%20runtime.mapdelete%E3%80%81mapdelete\_faststr%E3%80%81mapdelete\_fast32%20%E5%92%8C%20mapdelete\_fast64%EF%BC%8C%E4%B8%BB%E4%BD%93%E6%B5%81%E7%A8%8B%E4%B8%8Ekey%E7%9A%84%E6%8F%92%E5%85%A5%E7%B1%BB%E4%BC%BC%EF%BC%8C%E8%BF%99%E9%87%8C%E6%8C%91%E9%80%89runtime.mapdelete%E6%9D%A5%E5%88%86%E6%9E%90%E4%B8%80%E4%B8%8B%E3%80%82%0A%60%60%60%0Afunc%20mapdelete\(t%20\*maptype%2C%20h%20\*hmap%2C%20key%20unsafe.Pointer\)%20%7B%0A%0A%0A%09%2F%2F%E7%9C%81%E7%95%A5%E4%B8%80%E4%BA%9B%E5%89%8D%E7%BD%AE%E5%88%A4%E6%96%AD...%0A%0A%0A%09%2F%2F%E8%AE%A1%E7%AE%97key%E7%9A%84%E5%93%88%E5%B8%8C%E5%80%BC%0A%09hash%20%3A%3D%20t.hasher\(key%2C%20uintptr\(h.hash0\)\)%0A%0A%09h.flags%20%5E%3D%20hashWriting%0A%0A%09%2F%2F%E5%BE%97key%E5%AF%B9%E5%BA%94%E7%9A%84%E6%A1%B6%E5%9C%A8%E6%95%B0%E7%BB%84%E4%B8%AD%E7%9A%84%E4%BD%8D%E7%BD%AE%0A%09bucket%20%3A%3D%20hash%20%26%20bucketMask\(h.B\)%0A%0A%09%2F%2F%E5%A6%82%E6%9E%9C%E6%AD%A3%E5%B8%B8%E6%89%A9%E5%AE%B9%20%E5%88%99%E5%85%88%E6%90%AC%E8%BF%81%E4%B8%A4%E4%B8%AA%E6%A1%B6%0A%09if%20h.growing\(\)%20%7B%0A%09%09growWork\(t%2C%20h%2C%20bucket\)%0A%09%7D%0A%0A%09%2F%2F%E5%BE%97%E5%88%B0%E5%85%B7%E4%BD%93%E7%9A%84%E6%A1%B6%0A%09b%20%3A%3D%20\(\*bmap\)\(add\(h.buckets%2C%20bucket\*uintptr\(t.bucketsize\)\)\)%0A%09bOrig%20%3A%3D%20b%20%2F%2F%E5%AD%98%E4%B8%80%E4%B8%8B%20%E5%90%8E%E9%9D%A2%E5%8F%AF%E8%83%BD%E4%BC%9A%E7%94%A8%E5%88%B0%0A%09top%20%3A%3D%20tophash\(hash\)%0Asearch%3A%0A%09for%20%3B%20b%20\!%3D%20nil%3B%20b%20%3D%20b.overflow\(t\)%20%7B%20%20%2F%2F%E9%81%8D%E5%8E%86%E6%AD%A3%E5%B8%B8%E6%A1%B6%E5%8F%8A%E6%BA%A2%E5%87%BA%E6%A1%B6%0A%09%09for%20i%20%3A%3D%20uintptr\(0\)%3B%20i%20%3C%20bucketCnt%3B%20i%2B%2B%20%7B%20%2F%2F%E9%81%8D%E5%8E%86%E6%A1%B6%E7%9A%84%E6%AF%8F%E4%B8%AA%E6%A7%BD%0A%09%09%09if%20b.tophash%5Bi%5D%20\!%3D%20top%20%7B%20%2F%2F%E7%94%A8%E5%93%88%E5%B8%8C%E5%80%BC%E7%9A%84%E9%AB%988%E4%BD%8D%E6%9D%A5%E5%BF%AB%E9%80%9F%E8%AF%95%E9%94%99%20%0A%09%09%09%09if%20b.tophash%5Bi%5D%20%3D%3D%20emptyRest%20%7B%0A%09%09%09%09%09break%20search%0A%09%09%09%09%7D%0A%09%09%09%09continue%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%93%88%E5%B8%8C%E5%80%BC%E9%AB%988%E4%BD%8D%E7%9B%B8%E7%AD%89%20%E4%BB%8E%E6%A7%BD%E9%87%8C%E5%8F%96%E5%87%BAkey%20%E8%B7%9F%20%E4%BC%A0%E5%85%A5%E7%9A%84key%E5%81%9A%E5%AF%B9%E6%AF%94%0A%09%09%09k%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2Bi\*uintptr\(t.keysize\)\)%0A%09%09%09k2%20%3A%3D%20k%0A%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09k2%20%3D%20\*\(\(\*unsafe.Pointer\)\(k2\)\)%0A%09%09%09%7D%0A%09%09%09if%20\!t.key.equal\(key%2C%20k2\)%20%7B%0A%09%09%09%09continue%0A%09%09%09%7D%0A%09%09%09%0A%09%09%09%2F%2F%E6%B8%85%E9%99%A4key%E7%9A%84%E6%93%8D%E4%BD%9C%0A%09%09%09if%20t.indirectkey\(\)%20%7B%0A%09%09%09%09\*\(\*unsafe.Pointer\)\(k\)%20%3D%20nil%20%0A%09%09%09%7D%20else%20if%20t.key.ptrdata%20\!%3D%200%20%7B%20%20%2F%2F%E5%A6%82%E6%9E%9Ckey%E5%90%AB%E6%9C%89%E6%8C%87%E9%92%88%20%E6%B8%85%E9%99%A4key%0A%09%09%09%09memclrHasPointers\(k%2C%20t.key.size\)%20%0A%09%09%09%7D%0A%0A%0A%09%09%09%2F%2F%E6%B8%85%E9%99%A4value%E7%9A%84%E6%93%8D%E4%BD%9C%0A%09%09%09e%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2BbucketCnt\*uintptr\(t.keysize\)%2Bi\*uintptr\(t.elemsize\)\)%0A%09%09%09if%20t.indirectelem\(\)%20%7B%0A%09%09%09%09\*\(\*unsafe.Pointer\)\(e\)%20%3D%20nil%0A%09%09%09%7D%20else%20if%20t.elem.ptrdata%20\!%3D%200%20%7B%20%2F%2F%E5%A6%82%E6%9E%9Cvalue%E5%90%AB%E6%9C%89%E6%8C%87%E9%92%88%20%E6%B8%85%E9%99%A4value%0A%09%09%09%09memclrHasPointers\(e%2C%20t.elem.size\)%0A%09%09%09%7D%20else%20%7B%0A%09%09%09%09memclrNoHeapPointers\(e%2C%20t.elem.size\)%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E8%BF%99%E4%B8%AA%E6%A7%BD%E6%A0%87%E8%AE%B0%E4%B8%BA%E8%A2%AB%E6%B8%85%E9%99%A4%E8%BF%87%0A%09%09%09b.tophash%5Bi%5D%20%3D%20emptyOne%0A%0A%0A%09%09%09%2F%2F%E4%B8%8B%E9%9D%A2%E8%BF%99%E4%B8%80%E6%AE%B5%E6%98%AF%E5%8E%BB%E5%88%A4%E6%96%AD%E8%A6%81%E4%B8%8D%E8%A6%81%E5%B0%86%E5%BD%93%E5%89%8D%E7%9A%84%E6%A7%BD%E6%A0%87%E8%AE%B0%E4%B8%BAemptyRest%0A%09%09%09if%20i%20%3D%3D%20bucketCnt\-1%20%7B%0A%09%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E6%98%AF%E6%A1%B6%E7%9A%84%E6%9C%80%E5%90%8E%E4%B8%80%E4%B8%AA%E6%A7%BD%EF%BC%8C%E9%82%A3%E4%B9%88%E5%88%A4%E6%96%AD%E6%BA%A2%E5%87%BA%E6%A1%B6%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%E6%98%AF%E4%B8%8D%E6%98%AFemptyRest%EF%BC%8C%E5%A6%82%E6%9E%9C%E6%98%AF%E9%82%A3%E8%BF%99%E4%B8%AA%E6%A7%BD%E4%B9%9F%E5%B0%86%E6%A0%87%E8%AE%B0%E4%B8%BAemptyRest%0A%09%09%09%09if%20b.overflow\(t\)%20\!%3D%20nil%20%26%26%20b.overflow\(t\).tophash%5B0%5D%20\!%3D%20emptyRest%20%7B%0A%09%09%09%09%09goto%20notLast%0A%09%09%09%09%7D%0A%09%09%09%7D%20else%20%7B%0A%09%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E4%B8%8D%E6%98%AF%E6%A1%B6%E7%9A%84%E6%9C%80%E5%90%8E%E4%B8%80%E4%B8%AA%E6%A7%BD%EF%BC%8C%E5%88%A4%E6%96%AD%E8%AF%A5%E6%A1%B6%E7%9A%84%E4%B8%8B%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%E6%98%AF%E4%B8%8D%E6%98%AFemptyRest%EF%BC%8C%E5%A6%82%E6%9E%9C%E6%98%AF%E9%82%A3%E8%BF%99%E4%B8%AA%E6%A7%BD%E4%B9%9F%E5%B0%86%E6%A0%87%E8%AE%B0%E4%B8%BAemptyRest%0A%09%09%09%09if%20b.tophash%5Bi%2B1%5D%20\!%3D%20emptyRest%20%7B%0A%09%09%09%09%09goto%20notLast%0A%09%09%09%09%7D%0A%09%09%09%7D%0A%0A%09%09%09%2F%2F%E5%88%B0%E8%BF%99%E9%87%8C%E6%97%B6%E9%9C%80%E8%A6%81%E6%A0%87%E8%AE%B0%E5%BD%93%E5%89%8D%E6%A7%BD%E4%B8%BAemptyRest%20%E5%8D%B3%E8%AF%A5%E6%A7%BD%E5%90%8E%E9%9D%A2%E7%9A%84%E6%A7%BD%E9%83%BD%E6%B2%A1%E6%9C%89key%E4%BA%86%EF%BC%8C%E5%B9%B6%E4%B8%94%E8%AF%A5%E6%A7%BD%E6%89%80%E5%9C%A8%E7%9A%84%E6%A1%B6%E5%90%8E%E9%9D%A2%E7%9A%84%E6%BA%A2%E5%87%BA%E6%A1%B6%E4%B9%9F%E6%B2%A1%E6%9C%89%E6%95%B0%E6%8D%AE%E4%BA%86%0A%09%09%09%2F%2F%E5%B9%B6%E4%B8%94%E4%B8%80%E7%9B%B4%E5%BE%80%E5%89%8D%E6%9F%A5%E6%89%BE%E5%8F%AF%E4%BB%A5%E6%A0%87%E8%AE%B0%E4%B8%BAemptyRest%E7%9A%84%E6%A7%BD%0A%09%09%09for%20%7B%0A%09%09%09%09b.tophash%5Bi%5D%20%3D%20emptyRest%0A%0A%0A%09%09%09%09%2F%2F%E8%BF%99%E6%AE%B5if%2Felse%E6%98%AF%E6%89%BE%E5%BD%93%E5%89%8D%E6%A7%BD%E7%9A%84%E5%89%8D%E4%B8%80%E4%B8%AA%E4%BD%8D%E7%BD%AE%0A%09%09%09%09if%20i%20%3D%3D%200%20%7B%20%2F%2F%E5%BD%93%E5%89%8D%E6%A7%BD%E4%BD%8D%E6%98%AF%E8%AF%A5%E6%A1%B6%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%20%E9%9C%80%E8%A6%81%E6%89%BE%E5%89%8D%E4%B8%80%E4%B8%AA%E6%A1%B6%E7%9A%84%E5%80%92%E6%95%B0%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%0A%09%09%09%09%09if%20b%20%3D%3D%20bOrig%20%7B%0A%09%09%09%09%09%09break%20%2F%2F%E5%BD%93%E5%89%8D%E6%98%AF%E6%AD%A3%E5%B8%B8%E6%A1%B6%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%20%E6%B2%A1%E6%9C%89%E5%89%8D%E4%B8%80%E4%B8%AA%E4%BD%8D%E7%BD%AE%E4%BA%86%20%0A%09%09%09%09%09%7D%0A%09%09%09%09%09%0A%09%09%09%09%09%2F%2F%E6%89%BE%E8%AF%A5%E6%A1%B6%E7%9A%84%E5%89%8D%E4%B8%80%E4%B8%AA%E6%A1%B6%20%E5%9B%A0%E4%B8%BA%E6%AD%A3%E5%B8%B8%E6%A1%B6%E8%B7%9F%E6%BA%A2%E5%87%BA%E6%A1%B6%E4%B9%8B%E9%97%B4%E6%98%AF%E5%8D%95%E9%93%BE%E8%A1%A8%EF%BC%8C%E6%89%80%E4%BB%A5%E6%B2%A1%E6%B3%95%E7%9B%B4%E6%8E%A5%E9%80%9A%E8%BF%87%E5%BD%93%E5%89%8D%E6%A1%B6%E6%89%BE%E5%88%B0%E4%B8%8A%E4%B8%80%E4%B8%AA%E6%A1%B6%EF%BC%8C%E5%8F%AA%E8%83%BD%E4%BB%8E%E6%AD%A3%E5%B8%B8%E6%A1%B6bOrig%E5%BC%80%E5%A7%8B%E4%BB%8E%E5%A4%B4%E9%81%8D%E5%8E%86%0A%09%09%09%09%09c%20%3A%3D%20b%0A%09%09%09%09%09for%20b%20%3D%20bOrig%3B%20b.overflow\(t\)%20\!%3D%20c%3B%20b%20%3D%20b.overflow\(t\)%20%7B%0A%09%09%09%09%09%7D%0A%09%09%09%09%09i%20%3D%20bucketCnt%20\-%201%0A%09%09%09%09%7D%20else%20%7B%20%2F%2F%E5%BD%93%E5%89%8D%E6%A7%BD%E4%BD%8D%E4%B8%8D%E6%98%AF%E8%AF%A5%E6%A1%B6%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%20%E7%BB%A7%E7%BB%AD%E5%BE%80%E5%89%8D%E6%89%BE%E4%B8%8A%E4%B8%80%E4%B8%AA%E6%A7%BD%E4%BD%8D%0A%09%09%09%09%09i\-\-%0A%09%09%09%09%7D%0A%0A%0A%09%09%09%09%2F%2F%E5%A6%82%E6%9E%9C%E5%89%8D%E4%B8%80%E4%B8%AA%E4%BD%8D%E7%BD%AE%E4%B8%8D%E6%98%AF%E7%A9%BA%E6%A7%BD%EF%BC%88emptyOne%EF%BC%89%E9%82%A3%E6%A0%87%E8%AE%B0emptyRest%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%B0%B1%E7%BB%93%E6%9D%9F%E4%BA%86%0A%09%09%09%09if%20b.tophash%5Bi%5D%20\!%3D%20emptyOne%20%7B%0A%09%09%09%09%09break%0A%09%09%09%09%7D%0A%09%09%09%7D%0A%0A%0A%09%09notLast%3A%0A%09%09%09%2F%2F%E5%85%83%E7%B4%A0%E6%95%B0%E5%87%8F%E5%B0%91%0A%09%09%09h.count\-\-%0A%09%09%09%2F%2F%E5%A6%82%E6%9E%9Cmap%E6%B2%A1%E6%9C%89kv%E4%BA%86%20%E5%88%99%E9%87%8D%E7%BD%AE%E8%BF%99%E4%B8%AAmap%E7%9A%84%E5%93%88%E5%B8%8C%E5%9B%A0%E5%AD%90%0A%09%09%09if%20h.count%20%3D%3D%200%20%7B%0A%09%09%09%09h.hash0%20%3D%20fastrand\(\)%0A%09%09%09%7D%0A%09%09%09break%20search%0A%09%09%7D%0A%09%7D%0A%0A%09%2F%2F%E6%B8%85%E7%90%86%E5%86%99%E6%A0%87%E5%BF%97%E4%BD%8D%0A%09if%20h.flags%26hashWriting%20%3D%3D%200%20%7B%0A%09%09throw\(%22concurrent%20map%20writes%22\)%0A%09%7D%0A%09h.flags%20%26%5E%3D%20hashWriting%0A%7D%0A%60%60%60%0A%0A%0A%0A%23%23%23%23%20map%E7%9A%84%E9%81%8D%E5%8E%86%0A%E5%A6%82%E6%9E%9C%E5%8D%95%E7%BA%AF%E5%9C%B0%E9%81%8D%E5%8E%86map%E6%98%AF%E4%B8%80%E4%BB%B6%E6%AF%94%E8%BE%83%E7%AE%80%E5%8D%95%E7%9A%84%E4%BA%8B%EF%BC%8C%E6%9C%80%E5%A4%96%E5%B1%82%E9%81%8D%E5%8E%86%E6%89%80%E6%9C%89%E7%9A%84bucket\(%E5%8C%85%E6%8B%AColdbuckets\)%EF%BC%8C%E4%B8%AD%E9%97%B4%E9%81%8D%E5%8E%86bucket%E9%87%8C%E7%9A%84%E6%A7%BD\(%E5%8C%85%E6%8B%AC%E6%BA%A2%E5%87%BA%E6%A1%B6\)%EF%BC%8C%E5%8D%B3%E5%8F%AF%E8%8E%B7%E5%8F%96map%E9%87%8C%E7%9A%84%E6%89%80%E6%9C%89%E7%9A%84kv%E5%AF%B9%E3%80%82%E4%BD%86%E5%AE%9E%E9%99%85%E4%B8%8Amap%E7%9A%84%E9%81%8D%E5%8E%86%E5%B9%B6%E4%B8%8D%E6%98%AF%E6%9C%89%E5%BA%8F%E7%9A%84%EF%BC%8CGo%E5%9B%A2%E9%98%9F%E5%9C%A8%E8%AE%BE%E8%AE%A1%E5%93%88%E5%B8%8C%E8%A1%A8%E7%9A%84%E9%81%8D%E5%8E%86%E6%97%B6%E5%B0%B1%E4%B8%8D%E6%83%B3%E8%AE%A9%E4%BD%BF%E7%94%A8%E8%80%85%E4%BE%9D%E8%B5%96%E5%9B%BA%E5%AE%9A%E7%9A%84%E9%81%8D%E5%8E%86%E9%A1%BA%E5%BA%8F%EF%BC%8C%E6%89%80%E4%BB%A5%E5%BC%95%E5%85%A5%E4%BA%86%E9%9A%8F%E6%9C%BA%E6%95%B0%E4%BF%9D%E8%AF%81%E9%81%8D%E5%8E%86%E7%9A%84%E9%9A%8F%E6%9C%BA%E6%80%A7%E3%80%82%0A%E5%9C%A8%E9%81%8D%E5%8E%86map%E6%97%B6%EF%BC%8C%E7%BC%96%E8%AF%91%E5%99%A8%E4%BC%9A%E4%BD%BF%E7%94%A8%20runtime.mapiterinit%20%E5%92%8C%20runtime.mapiternext%20%E4%B8%A4%E4%B8%AA%E8%BF%90%E8%A1%8C%E6%97%B6%E5%87%BD%E6%95%B0%E9%87%8D%E5%86%99%E5%8E%9F%E5%A7%8B%E7%9A%84%20for\-range%20%E5%BE%AA%E7%8E%AF%E3%80%82%0A%0Aruntime.mapiterinit%E4%BC%9A%E5%88%9D%E5%A7%8B%E5%8C%96%E9%81%8D%E5%8E%86%E5%BC%80%E5%A7%8B%E7%9A%84%E5%85%83%E7%B4%A0%E3%80%82%E5%85%B6%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91%E5%A6%82%E4%B8%8B%E3%80%82%0A%60%60%60%0Aruntime.mapiterinit%E7%9A%84%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91%E5%A6%82%E4%B8%8B%E3%80%82%0Afunc%20mapiterinit\(t%20\*maptype%2C%20h%20\*hmap%2C%20it%20\*hiter\)%20%7B%0A%0A%09...%0A%0A%09it.t%20%3D%20t%0A%09it.h%20%3D%20h%0A%09it.B%20%3D%20h.B%0A%09it.buckets%20%3D%20h.buckets%0A%0A%0A%09if%20t.bucket.ptrdata%20%3D%3D%200%20%7B%0A%09%09h.createOverflow\(\)%0A%09%09it.overflow%20%3D%20h.extra.overflow%0A%09%09it.oldoverflow%20%3D%20h.extra.oldoverflow%0A%09%7D%0A%0A%09%2F%2F%20%E9%9A%8F%E6%9C%BA%E5%9B%A0%E5%AD%90%0A%09r%20%3A%3D%20uintptr\(fastrand\(\)\)%0A%09if%20h.B%20%3E%2031\-bucketCntBits%20%7B%0A%09%09r%20%2B%3D%20uintptr\(fastrand\(\)\)%20%3C%3C%2031%0A%09%7D%0A%09%2F%2F%20%E6%A0%B9%E6%8D%AE%E9%9A%8F%E6%9C%BA%E5%9B%A0%E5%AD%90%E8%AE%A1%E7%AE%97%20%E8%B5%B7%E5%A7%8B%E4%BD%8D%E7%BD%AE%0A%09it.startBucket%20%3D%20r%20%26%20bucketMask\(h.B\)%0A%09it.offset%20%3D%20uint8\(r%20%3E%3E%20h.B%20%26%20\(bucketCnt%20\-%201\)\)%0A%0A%09%2F%2F%20iterator%20state%0A%09it.bucket%20%3D%20it.startBucket%0A%0A%0A%09mapiternext\(it\)%0A%7D%0A%60%60%60%0A%0Aruntime.mapiternext%E8%B4%9F%E8%B4%A3%E9%81%8D%E5%8E%86%E5%85%83%E7%B4%A0%E3%80%82%E5%85%B6%E4%B8%AD%E5%8F%88%E5%88%86%E4%B8%BA%3Cu%3E%E9%80%89%E6%8B%A9%E6%A1%B6%3C%2Fu%3E%E4%B8%8E%3Cu%3E%E9%81%8D%E5%8E%86%E6%A1%B6%E5%86%85%E5%85%83%E7%B4%A0%3C%2Fu%3E%E4%B8%A4%E4%B8%AA%E9%98%B6%E6%AE%B5%E3%80%82%E5%85%B6%E6%A0%B8%E5%BF%83%E9%80%BB%E8%BE%91%E5%A6%82%E4%B8%8B%EF%BC%9A%0A%60%60%60%0Afunc%20mapiternext\(it%20\*hiter\)%20%7B%0A%09h%20%3A%3D%20it.h%0A%09t%20%3A%3D%20it.t%0A%09bucket%20%3A%3D%20it.bucket%0A%09b%20%3A%3D%20it.bptr%0A%09i%20%3A%3D%20it.i%0A%09checkBucket%20%3A%3D%20it.checkBucket%0A%0Anext%3A%0A%09if%20b%20%3D%3D%20nil%20%7B%0A%09%09if%20bucket%20%3D%3D%20it.startBucket%20%26%26%20it.wrapped%20%7B%0A%09%09%09it.key%20%3D%20nil%0A%09%09%09it.elem%20%3D%20nil%0A%09%09%09return%0A%09%09%7D%0A%0A%09%09%2F%2F%E5%A6%82%E4%BD%95%E6%A1%B6%E4%B8%BA%E7%A9%BA%20%E6%89%BE%E9%9C%80%E8%A6%81%E9%81%8D%E5%8E%86%E7%9A%84%E6%96%B0%E6%A1%B6%0A%09%09if%20h.growing\(\)%20%26%26%20it.B%20%3D%3D%20h.B%20%7B%0A%09%09%09oldbucket%20%3A%3D%20bucket%20%26%20it.h.oldbucketmask\(\)%0A%09%09%09b%20%3D%20\(\*bmap\)\(add\(h.oldbuckets%2C%20oldbucket\*uintptr\(t.bucketsize\)\)\)%0A%09%09%09if%20\!evacuated\(b\)%20%7B%0A%09%09%09%09checkBucket%20%3D%20bucket%0A%09%09%09%7D%20else%20%7B%0A%09%09%09%09b%20%3D%20\(\*bmap\)\(add\(it.buckets%2C%20bucket\*uintptr\(t.bucketsize\)\)\)%0A%09%09%09%09checkBucket%20%3D%20noCheck%0A%09%09%09%7D%0A%09%09%7D%20else%20%7B%0A%09%09%09b%20%3D%20\(\*bmap\)\(add\(it.buckets%2C%20bucket\*uintptr\(t.bucketsize\)\)\)%0A%09%09%09checkBucket%20%3D%20noCheck%0A%09%09%7D%0A%09%09bucket%2B%2B%0A%09%09if%20bucket%20%3D%3D%20bucketShift\(it.B\)%20%7B%0A%09%09%09bucket%20%3D%200%0A%09%09%09it.wrapped%20%3D%20true%0A%09%09%7D%0A%09%09i%20%3D%200%0A%09%7D%0A%0A%09%2F%2F%E5%89%A9%E4%B8%8B%E7%9A%84%E9%83%A8%E5%88%86%E5%B0%B1%E6%98%AF%E9%81%8D%E5%8E%86%E6%A1%B6%E7%9A%84%E5%85%83%E7%B4%A0%EF%BC%8C%E4%B8%8D%E8%BF%87%E5%A6%82%E6%9E%9C%E5%93%88%E5%B8%8C%E8%A1%A8%E5%A4%84%E4%BA%8E%E6%89%A9%E5%AE%B9%E6%9C%9F%E9%97%B4%E5%B0%B1%E4%BC%9A%E8%B0%83%E7%94%A8%20runtime.mapaccessK%20%E8%8E%B7%E5%8F%96%E9%94%AE%E5%80%BC%E5%AF%B9%EF%BC%9A%0A%09%09for%20%3B%20i%20%3C%20bucketCnt%3B%20i%2B%2B%20%7B%0A%09%09offi%20%3A%3D%20\(i%20%2B%20it.offset\)%20%26%20\(bucketCnt%20\-%201\)%0A%09%09k%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2Buintptr\(offi\)\*uintptr\(t.keysize\)\)%0A%09%09v%20%3A%3D%20add\(unsafe.Pointer\(b\)%2C%20dataOffset%2BbucketCnt\*uintptr\(t.keysize\)%2Buintptr\(offi\)\*uintptr\(t.valuesize\)\)%0A%09%09if%20\(b.tophash%5Boffi%5D%20\!%3D%20evacuatedX%20%26%26%20b.tophash%5Boffi%5D%20\!%3D%20evacuatedY\)%20%7C%7C%0A%09%09%09\!\(t.reflexivekey\(\)%20%7C%7C%20alg.equal\(k%2C%20k\)\)%20%7B%0A%09%09%09it.key%20%3D%20k%0A%09%09%09it.value%20%3D%20v%0A%09%09%7D%20else%20%7B%0A%09%09%09rk%2C%20rv%20%3A%3D%20mapaccessK\(t%2C%20h%2C%20k\)%0A%09%09%09it.key%20%3D%20rk%0A%09%09%09it.value%20%3D%20rv%0A%09%09%7D%0A%09%09it.bucket%20%3D%20bucket%0A%09%09it.i%20%3D%20i%20%2B%201%0A%09%09return%0A%09%7D%0A%09b%20%3D%20b.overflow\(t\)%0A%09i%20%3D%200%0A%09goto%20next%0A%7D%0A%0A%0A%60%60%60%0A%0A
