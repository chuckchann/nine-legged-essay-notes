# slice

## 简介

本文旨在讲解slice的数据结果及扩容策略，以下内容基于：

```shell
go version go1.16.2 darwin/amd64
```

## slice数据结构

```go
type slice struct {
	array unsafe.Pointer  //指向底层数组的指针
  len   int   //slice的长度 len(s)的返回值 for range 遍历slice的话也是以这个为终点
  cap   int   //slice的容量 cap(s)的返回值
}
```

## slice扩容策略

可以使用append函数给slice添加元素

```go
	var s []int
	s = append(s, 1, 2, 3)
	fmt.Println(s)
```

当slice的容量不足的时候就会进行扩容，扩容调用的是runtime.growslice函数

```go
func growslice(et *_type, old slice, cap int) slice {

  //省略部分前置判断代码。。。
  
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap { //所需要的容量大于两倍的旧容量 那么 新容量(newcap) = 所需要的容量
		newcap = cap
	} else { //所需要的容量小于两倍的旧容量
		if old.cap < 1024 {
			newcap = doublecap //如果旧容量小于1024 那么 新容量(newcap) = 2*旧容量
		} else {
      //如果旧容量(newcap)大于1024 那么每次增长旧容量(newcap)的1/4，直到 旧容量(newcap) >= 所需要的容量  此时的旧容量就为新slice的容量
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}

			if newcap <= 0 {
				newcap = cap
			}
		}
	}
  
  //省略部分代码。。。

  //将旧的slcie的底层数组迁移到新的slice中
	memmove(p, old.array, lenmem)

  //使用新容量直接赋值给新的slice
	return slice{p, old.len, newcap}
}
```

扩容策略总结：

- 所需要的容量大于两倍的旧容量，那么新容量直接等于所需的容量
- 所需要的容量小于两倍的旧容量，再判断：
  - 旧容量小于1024，新容量直接等于两倍的旧容量
  - 旧容量大于1024，逐步扩容，每次都只增加旧容量的1/4，直到 旧容量 >= 所需要的容量，此时的扩容后的旧容量就为新容量
