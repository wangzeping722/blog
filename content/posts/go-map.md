---
title: "go map 的实现"
date: 2021-04-24T08:22:17+08:00
draft: true
---

golang 中的 map 本质上就是一个 hash table。所有的数据都存储的一个 buckets 的数组中，每个 bucket 都包含 8 个键值对。计算 key 的 hash 值，然后通过 hash 值来判断把 data 存储在哪里。在如何安置元素上，有两个核心的点：

- key hash 的低位用来判断 data 应该放置在 buckets 的哪个 bucket 中
- key hash 的高位用来判断 data 具体放在 bucket 的哪个位置

需要注意的问题：

- 当在使用 range 访问 map 的时候，不会去移动当前 bucket 中的 key，这样会导致data 出现 0 次或 2 两次。

### hmap

`hmap` 是 go map 的底层数据结构，其定义如下：

``` go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

  // buckets 存储元素
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0. 
  // oldbuckets 在扩容阶段存储元素
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```

- `count` 表示当前哈希表中的元素数量；
- `hash0` 是哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入；
- `oldbuckets` 是哈希在扩容时用于保存之前 `buckets` 的字段，它的大小是当前 `buckets` 的一半；

buckets 是一个 Pointer 指针，其实指向的是一个 `bmap` 的数组：

``` go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```



### 创建一个 map

golang 中创意一个 map 是调用的 make() 这个内置函数来创建的，因为 map 的创建可以进行优化，go 在编译期会根据数据类型和大小的的不同来选择不同的 makemap 函数，不过最终都是调用了 makemap

接下来我们具体分析一下 makemap：

``` go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // 计算哈希占用的内存是否溢出或者超出能分配的最大值；
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```

makemap 主要做了以下的事情：

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 fastrand() 函数分配一个随机种子；
3. 根据传入的 hint 值，计算出所需要的最小的 bucket 数量
4. 调用 `makeBucketArray` 分配初始化的 hash 表（如果 B == 0， 这里有一个懒加载的策略）



### 如何查找元素

mapaccess1   ---- 返回元素
mapaccess2   ---- 返回元素的同时，还会判断 k-v 是否存在



### 如何设置元素

```
mapassign
```

1. 查找 key 处于的 bucket
2. 判断是否 map 处于扩容状态（算法用的是渐进式 rehash）
   1. 调用 `growWork` 做一次扩容操作
      1. evacuate 旧的 bucket 中key对应的 bucket到新的 bucket
      2. 再调用一次 evacuate

### 如何删除元素



### 如何扩容

触发扩容的 loadFactor 是 6.5

### 如何缩容



buckets 以增量的方式从旧的 bucket 拷贝到新的 bucket 上。





Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 `tophash` 就成为可以帮助哈希快速遍历桶中元素的缓存。

哈希表的每个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会存储到哈希的溢出桶中。随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。