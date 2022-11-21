# blotdb

------

## 1. 简介

**boltdb是一个纯go编写的支持事务的文件型单机kv数据库。** 其具有以下特点：

- 单机部署：不需要考虑**CAP**
- 支持事物：仅允许**多个只读事务和最多一个读写事务**同时运行
- 索引结构：因为是kv型的数据库，所以天然地只有主键索引（**B+树实现**），减少了磁盘IO
- 缓存管理：仅管理写缓存，利用**mmap**管理读缓存

------

## 2. 核心数据结构分析

boltdb是一款文件型数据库，即它的数据全部都是存储在文件上，下面来分析下它的在数据是如何存储在文件中的。

### <u>*1. db*</u>

在boltdb中，一个db对应一个真实的磁盘文件。

```go
func main()  {
	db, err := bolt.Open("my.db", 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```

执行完上面的代码后，可以看到根目录多一个my.db的文件

```shell
ll

total 40
-rw-r--r--  1 chuckchen  staff   173B Nov 16 10:56 main.go
-rw-------  1 chuckchen  staff    16K Nov 16 10:57 my.db

```

### <u>*2. page*</u>

在具体的磁盘文件中，boltdb又是以page为单位来读取和写入数据库的，即数据在磁盘上是以page为单位进行存储的，page的大小与操作系统保持一致（一般为4k）。

其中每个page又由 **页头数据** + **真实数据** 组成，页的数据结构如下：

![storage-and-cache-page-layout](image/storage-and-cache-page-layout.jpg)

page的数据定义如下：

```go
//page.go

type pgid uint64

type page struct {
  //page header 
	id       pgid    //页id
	flags    uint16  //标记位，用来表示页的类型，枚举下面再介绍
	count    uint16  //个数，用来记录page中元素的个数
	overflow uint32  //当遇到体积巨大，单个page无法装下的数据时，会溢出到其他paage，overflow记录溢出总数
  
  //page body
	ptr      uintptr //指向page数据的内存地址，该字段仅在内存中存在
}
```

page类型的枚举值如下：

```go
//page.go

const (
  branchPageFlag   = 0x01  //分支节点页 存储索引信息(页号、元素key值)，即B+树的非叶子节点
  leafPageFlag     = 0x02  //叶子节点页 存储数据信息(页号，kv值)，即B+树的
  metaPageFlag     = 0x04  //存储数据库的元信息(空闲列表页id、放置桶的根页等)
	freelistPageFlag = 0x10  //存储哪些页是空闲页，可以用来后续分配空间时，优先考虑分配
)
```

下面再具体看看每种page类型分别有什么作用。

#### <u>*2.1. meta*</u>

meta page 记录 Bolt 实例的所有元数据，它告诉用户**这是什么文件**以及**如何解读这个文件**，具体结构如下：

![storage-and-cache-meta-page-layout](image/storage-and-cache-meta-page-layout.jpg)

```go
//db.go

type meta struct {
	magic    uint32 //随机数，用来确定该文件是一个bolt数据库文件(另一种文件起始位置拥有相同数据的可能性极低)
	version  uint32 //表明该文件所属的bolt版本，便于日后做兼容与迁移
	pageSize uint32 //页大小 与操作系统一致
	flags    uint32 //保留字段，未使用
	root     bucket //bolt实例所有索引和原数据被组织成一个树形结构，root 就是根节点
	freelist pgid   //bolt删除数据时可能出现富余的空间，这些空间会被记录在freelist中备用
	pgid     pgid   //下一个要被分配的 page 的 id，取值大于已分配的所有 pages 的 id
	txid     txid   //下一个要被分配的事务 id。事务 id 单调递增，可以被理解为事务执行的逻辑时间
	checksum uint64 //用于确认 meta page 数据本身的完整性，保证读取的是上一次写入的数据
}
```

将meta写入page代码如下：

```go
func (m *meta) write(p *page) {
	if m.root.root >= m.pgid {
		panic(fmt.Sprintf("root bucket pgid (%d) above high water mark (%d)", m.root.root, m.pgid))
	} else if m.freelist >= m.pgid {
		panic(fmt.Sprintf("freelist pgid (%d) above high water mark (%d)", m.freelist, m.pgid))
	}

  //page id 要么是0，要么是1  这取决于 meta.txid
	p.id = pgid(m.txid % 2)
	//设置page类型
  p.flags |= metaPageFlag

	// 计算checksum
	m.checksum = m.sum64()

  //将met拷贝到page.ptr
	m.copy(p.meta())
}
```

#### <u>*2.2 freelist*</u>

freelist 记录着一个**有序**的 page id 列表：

![storage-and-cache-freelist-page-layout](image/storage-and-cache-freelist-page-layout.jpg)

freelist的数据结构如下：

```go
//freelist.go

type freelist struct {
	ids     []pgid          // 可以用来分配的空闲页id
	pending map[txid][]pgid // 将来很快能被释放的空闲页，部分事务可能在读或在写
	cache   map[pgid]bool   // 一个map，用来快速判断某个页是否为空闲页
}
```

将空闲列表转换成页信息，代码如下：

```go
// write writes the page ids onto a freelist page. All free and pending ids are
// saved to disk since in the event of a program crash, all pending ids will
// become free.
func (f *freelist) write(p *page) error {
	// Combine the old free pgids and pgids waiting on an open transaction.

	// Update the header flag.
	p.flags |= freelistPageFlag

  //page.count是一个uint16类型的数字，这也意味着它的最大值只能到 2^16 = 65536, 如果空闲页的个数
  //超过65535，则需要将page.prt中的第一个字节来存储其他的空闲页个数，同时将p.count设置为0xFFFF
	lenids := f.count()   //计算空闲列表的总数
	if lenids == 0 {
		p.count = uint16(lenids)
	} else if lenids < 0xFFFF { // 不超过65535 
		p.count = uint16(lenids)
		f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[:])
	} else { //超过65535
		p.count = 0xFFFF
		((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0] = pgid(lenids)
		f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[1:])
	}

	return nil
}
```

todo...

#### <u>*2.3. branch*</u>

bolt 利用 B+ 树存储索引和键值数据本身，这里的 branch 指的就是 B+ 树的分支节点。 branch 需要存储大小不同键值数据，布局如下：

![storage-and-cache-branch-page-layout](image/storage-and-cache-branch-page-layout.jpg)

分支节点在存储时，一个分支节点页会存储多个分支页元素，即branchPageElement，这个信息可以记作为分支页元素元信息。元信息中定义了具体该元素的页(pgid)、改元素所指向的页中

```go
type branchPageElement struct {
	pos   uint32 //该元信息与真实key之间的偏移
	ksize uint32 //key的长度
	pgid  pgid //该元素的页id
}
```

Todo...

#### <u>*2.4 leaf*</u>

叶子节点主要用来存储实际的数据，leaf里存的是key + value，其布局如下：

![storage-and-cache-leaf-page](image/storage-and-cache-leaf-page.jpg)

```go
//page.go

type leafPageElement struct {
	flags uint32 //用来判断是子桶叶子节点元素还是普通的k/v叶子节点元
	pos   uint32 //偏移位置
	ksize uint32 //key的大小
	vsize uint32 //value的大小
}

//叶子节点的key
func (n *leafPageElement) key() []byte {
	buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
  //pos - pos+ksize
	return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos]))[:n.ksize:n.ksize]
}

//叶子节点的value
func (n *leafPageElement) value() []byte {
	buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
  //pos+ksize - pos+ksize+vsize
	return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos+n.ksize]))[:n.vsize:n.vsize]
}

//从page里提取某个具体的叶子节点元素
func (p *page) branchPageElement(index uint16) *branchPageElement {
	return &((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[index]
}

//从page里提取所有的叶子节点元素
func (p *page) leafPageElements() []leafPageElement {
	if p.count == 0 {
		return nil
	}
	return ((*[0x7FFFFFF]leafPageElement)(unsafe.Pointer(&p.ptr)))[:]
}
```

### <u>*3. node*</u>

在**磁盘**中数据是用**page**来表示的，但是在**内存**中数据是用**node**来表示的，node可以说是解序列化后的page。

```go
//node.go

type node struct {
	bucket     *Bucket
	isLeaf     bool  //区分是分支节点还是叶子节点 
	unbalanced bool  //
	spilled    bool
	key        []byte
	pgid       pgid
	parent     *node
	children   nodes //子节点 如果是叶子节点这个字段为空
	inodes     inodes //inode表示内部节点 inodes表示改节点存储的若干个kv值 inode会在 B+ 树中进行路由——二分查找时使用。
}


type nodes []*node

func (s nodes) Len() int           { return len(s) }
func (s nodes) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }
func (s nodes) Less(i, j int) bool { return bytes.Compare(s[i].inodes[0].key, s[j].inodes[0].key) == -1 }

type inode struct {
	flags uint32  // 表示是否是子桶叶子节点还是普通叶子节点。如果flags值为1表示子桶叶子节点，否则为普通叶子节点
	pgid  pgid  //当inode为分支元素时，pgid才有值，为叶子元素时，则没值
	key   []byte
	value []byte //当inode为分支元素时，value为空，为叶子元素时，才有值
}

type inodes []inode


//将page转换为内存中的node
func (n *node) read(p *page) {
	n.pgid = p.id
	n.isLeaf = ((p.flags & leafPageFlag) != 0)
	n.inodes = make(inodes, int(p.count))

	for i := 0; i < int(p.count); i++ {
		inode := &n.inodes[i]
		if n.isLeaf { //如果是叶子节点
      //获取叶子节点元素
			elem := p.leafPageElement(uint16(i))
      //子桶 or 叶子节点
			inode.flags = elem.flags
      //读取k/v
			inode.key = elem.key()
			inode.value = elem.value()
		} else {//如果是分支节点
      //获取分支节点元素
			elem := p.branchPageElement(uint16(i))
      //记录页id
			inode.pgid = elem.pgid
      //读取k
			inode.key = elem.key()
		}
		_assert(len(inode.key) > 0, "read: zero-length inode key")
	}

	// Save first key so we can find the node in the parent when we spill.
	if len(n.inodes) > 0 {
    //将该页的第一个key放入node.key中，以便父节点以此作为索引进行查找和路由
		n.key = n.inodes[0].key
		_assert(len(n.key) > 0, "read: zero-length node key")
	} else {
		n.key = nil
	}
}

//将node转换为page
func (n *node) write(p *page) {
	// Initialize page.
  
  //判断是叶子节点还是分支节点
	if n.isLeaf {
		p.flags |= leafPageFlag
	} else {
		p.flags |= branchPageFlag
	}

	if len(n.inodes) >= 0xFFFF {
		panic(fmt.Sprintf("inode overflow: %d (pgid=%d)", len(n.inodes), p.id))
	}
	p.count = uint16(len(n.inodes))

	// Stop here if there are no items to write.
	if p.count == 0 {
		return
	}

	// Loop over each item and write it to the page.
	b := (*[maxAllocSize]byte)(unsafe.Pointer(&p.ptr))[n.pageElementSize()*len(n.inodes):]
  
  //遍历indoes 将inode信息写入page.ptr中
	for i, item := range n.inodes {
		_assert(len(item.key) > 0, "write: zero-length inode key")

		// Write the page element.
		if n.isLeaf {
      //构建一个page的叶子节点元素
			elem := p.leafPageElement(uint16(i))
      //将相关字段信息写入元素中
			elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))
			elem.flags = item.flags
			elem.ksize = uint32(len(item.key))
			elem.vsize = uint32(len(item.value))
		} else {
      //构建一个page的分支节点元素
			elem := p.branchPageElement(uint16(i))
      //将相关字段信息写入元素中
			elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))
			elem.ksize = uint32(len(item.key))
			elem.pgid = item.pgid
			_assert(elem.pgid != p.id, "write: circular dependency occurred")
		}
    
    //如果kv值太大 超过了剩下的可分配的空间，那么需要重新分配空间
		klen, vlen := len(item.key), len(item.value)
		if len(b) < klen+vlen {
			b = (*[maxAllocSize]byte)(unsafe.Pointer(&b[0]))[:]
		}

		//将数据（kv）放到 page的body里 
		copy(b[0:], item.key)
		b = b[klen:]
		copy(b[0:], item.value)
		b = b[vlen:]
	}

	// DEBUG ONLY: n.dump()
}



```

### <u>*4. bucket*</u>

bucket是一堆kv对的集合，如果**db**代表关系型数据库中的**"库"**，那么**bucket**则代表关系型数据库中的**"表"**。bucket还支持无限嵌套，即bucket里可以创建子bucket，为开发者归类数据提供更灵活的方案。一个bucket代表一颗完整的b+树。

```go
//bucket.go

type Bucket struct {
	*bucket                     // 内联的bucket
	tx       *Tx                // 关联的事物 事务后面再讲 先忽略
	buckets  map[string]*Bucket // subbucket cache
	page     *page              // inline page reference
	rootNode *node              // materialized node for the root page.
	nodes    map[pgid]*node     // node cache

	// Sets the threshold for filling nodes when they split. By default,
	// the bucket will fill to 50% but it can be useful to increase this
	// amount if you know that your write workloads are mostly append-only.
	//
	// This is non-persisted across transactions so it must be set in every Tx.
  
  //填充率
	FillPercent float64
}

type bucket struct {
	root     pgid   // page id of the bucket's root-level page
	sequence uint64 // monotonically incrementing, used by NextSequence()
}
```

在初始化新的实例时bolt会创建根桶，在db.init()中可以看到

```GO
func (db *DB) init() error {
	// Set the page size to the OS page size.
	db.pageSize = os.Getpagesize()

	// Create two meta pages on a buffer.
	buf := make([]byte, db.pageSize*4)
	for i := 0; i < 2; i++ {
		...
	}

	// Write an empty freelist at page 3.
	p := db.pageInBuffer(buf[:], pgid(2))
	p.id = pgid(2)
	p.flags = freelistPageFlag
	p.count = 0

	// paid = 3 的这一页用来存放
	p = db.pageInBuffer(buf[:], pgid(3))
	p.id = pgid(3)
	p.flags = leafPageFlag
	p.count = 0

	// Write the buffer to our data file.
	if _, err := db.ops.writeAt(buf, 0); err != nil {
		return err
	}
	if err := fdatasync(db); err != nil {
		return err
	}

	return nil
}
```



