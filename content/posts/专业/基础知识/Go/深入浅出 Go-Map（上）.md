---
title: 深入浅出 Go-Map（上）
date: 2025-10-29T02:23:00+08:00
lastmod: 2025-10-29T02:23:00+08:00
draft: false
tags:
  - Golang
  - map
  - Hash
  - 计算机基础
categories:
author: 小石堆
---

    1.24 前后 map 的世界，在 1.24 后，go 的 map 换了实现方式，让我们来康康

## 1 追踪 map 的起始点

看下面这段代码，我们来定位一下 make 之后发生了什么？

```go
package main

func main() {
	_ = make(map[string]int, 16)
}
```

使用 `go tool compile -S` 编译文件，可以发现 map 实际上是定位到了底层的 runtime.makemap 中，这使得我们有了窥探 map 源码的机会。

```shell
❯ go tool compile -S main.go | grep 'make'
    0x0034 00052 (.../main.go:4)        CALL    runtime.makemap(SB)
    rel 52+4 t=R_CALLARM64 runtime.makemap+0
```

在进入具体方法之前，我们先了解一下 1.24 之前 go 的 map 的结构

## 2 1.24 前的世界

    本节 go 的代码是基于 1.23.2 版本的

### 2.1 map 的基本构造

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251029174955534.png)
在 1.24 前，go 的 map 实现是基于桶数组 + 溢出桶实现的，具体而言是基于 runtime 下的这两个结构体实现的。  
首先是 hmap，他是宏观上的 map，包含了 map 的键值对数量、状态、桶的数量以及桶的指针等等。当我们创建了一个 map 的时候，宏观来看就是创建 hmap。

```go
type hmap struct {
	// map 中键值对的数量
	count     int
	// 映射到上面的 const 中，用来标记当前 map 的状态
	flags     uint8
	// 桶的数量 数量为 2^B
	B         uint8
	noverflow uint16
	// 随机 hash 种子
	hash0     uint32 // hash seed
	// unsafe.Pointer 是指向任意地址的一个指针
	// 当前桶数组
	buckets    unsafe.Pointer
	// 旧桶数组
	oldbuckets unsafe.Pointer
	// 记录当前已经迁移完的旧桶数量 nevacuate = 2 ^ (B - 1) 时迁移完毕
	nevacuate  uintptr
	clearSeq   uint64
	extra *mapextra // optional fields
}
```

微观而言，让我们来看下面这个结构体。这个结构体是什么？这个结构体就是一个溢出桶，这里很奇怪的就是明明溢出桶里应该存放的是 8 个 KV 对，怎么这里只有一个 tophash 的数组？  
![](https://img.xiaoshidui.top/blog-pic/images/20251030222416534.png)
原因如下：**go 早期是不支持泛型的，当你创建任意类型的 map 的时候，需要编译器去做类型判断才能内存分配（如果用 `interface{}` 会怎么样？—— 文章末尾揭晓彩蛋），所以当编译器在推断出传入的类型后，会在这个结构体的内存后紧挨的规划出一片内存，用来存放 Key、Val 以及溢出桶指针。虽然整块内存都属于 bmap，但是 bmap 这个结构体只表述了头部的一部分内存，实际上的内存由编译器决定。**

```go
// A bucket for a Go map.
type bmap struct {
	// abi.OldMapBucketCount = 1 >> 3
	// hash 值的高八位
	tophash [abi.OldMapBucketCount]uint8
}
// 扩展后，可以推断出逻辑上的 bmap 长下面这样
type bmap struct {
	tophash [abi.OldMapBucketCount]uint8
	key [abi.OldMapBucketCount]keyType
	val [abi.OldMapBucketCount]valueType
	nextBmap *bmap
}
```

接着是 extra 字段

```
type mapextra struct {
	// 存放当前 map 下所有的溢出桶的指针
	overflow    *[]*bmap
	// 扩容迁移时，旧的溢出桶指针存放地址
	oldoverflow *[]*bmap
	// 额外创建一些 bmap 缓存起来供给
	nextOverflow *bmap
}
```

### 2.2 map 的常规操作

下面我们从四个 map 的常见操作走进基于 桶 + 溢出桶 的实现。

#### 2.2.1 创建

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030225101066.png)

当我们创建一个 map 的时候，殊途同归都会走向下面 `runtime.makemap()` 的方法。不妨追踪一下

```go
func main() {
	_ = make(map[string]int, 8)
}
```

使用 dlv 工具可以看到最后走向了 `runtime.makemap`，我们还可以看到具体的传参，参数具体的细节我们后面再说。

```shell
❯ dlv debug main.go
Type 'help' for list of commands.
(dlv) break runtime.makemap
Breakpoint 1 set at 0x102848a7c for runtime.makemap() /usr/local/go/src/runtime/map.go:318
(dlv) continue
> [Breakpoint 1] runtime.makemap() /usr/local/go/src/runtime/map.go:318 (hits goroutine(1):1 total:1) (PC: 0x102848a7c)
Warning: debugging optimized function
   313: //
   314: // Do not remove or change the type signature.
   315: // See go.dev/issue/67401.
   316: //
   317: //go:linkname makemap
=> 318: func makemap(t *maptype, hint int, h *hmap) *hmap {
   319:         mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
   320:         if overflow || mem > maxAlloc {
   321:                 hint = 0
   322:         }
   323:
(dlv) args
t = ("*internal/abi.MapType")(0x102867ac0)
hint = 16
h = (*runtime.hmap)(0x14000052728)
~r0 = (unreadable empty OP stack)
```

既然定位到了具体的方法，就让我们继续往下看方法的实现吧。

```go
//go:linkname makemap
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = uint32(rand())

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

参数 1： t \*mapType 实际上是一个结构体，他主要包含了 map 本身的类型、键/值的类型、桶的类型、Key 类型占据的字节数、Val 类型占据的字节数以及 一个 Bucket 整体占据的字节数等。多说无益，我们可以打印出来分析一下。
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251029190746936.png)
我们可以主要关注一下 KeySize、ValueSize 以及 BucketSize，他们分别对应了 String 的大小(16B)、Int 的大小(8B)，以及给一个 bucket 分配的内存的大小((8 + 16 _ 8 + 8 _ 8 + 8) B)。
参数 2：hint int 是指当我们使用 make 创建 map 时传入的容量提示

```shell
(dlv) print hint
16
```

参数 3：h \*hmap 实际上是一个可选的目标 hmap，调用方可以传入 hmap 对象给函数使用，因为我们的 map 本身就是一个 hmap 嘛。

```shell
(dlv) print *h
runtime.hmap {
        count: 0,
        flags: 0,
        B: 0,
        noverflow: 0,
        hash0: 0,
        buckets: unsafe.Pointer(0x0),
        oldbuckets: unsafe.Pointer(0x0),
        nevacuate: 0,
        extra: *runtime.mapextra nil,}
```

（1）首先会通过 hint \* 桶的大小，初步判断**有没有可能出现一次分配的内存 > 预计内存上界**，如果超过，会将 hint 置零

```go
mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
if overflow || mem > maxAlloc {
	hint = 0
}
```

（2）初始化 hmap 和 哈希种子，设定 **hash0** **随机种子**（每个 map 不同），抵抗哈希碰撞攻击，哈希函数会结合这个种子。

```go
if h == nil {
	h = new(hmap)
}
h.hash0 = uint32(rand())
```

（3）决定初始桶指数 B，我们知道 hamp 下面的桶数组是 2^B 大小，也就是说有 2^B 个溢出桶链表。overLoadFactor(hint, B) 是找到不超过 HashMap 负载因子（防止 Hash 劣化）的 B 的最小值。

```go
B := uint8(0)
for overLoadFactor(hint, B) {
    B++
}
h.B = B
```

（4）分配桶数组，当 B 等于 0 的时候懒分配（即插入的时候再分配），否则直接分配桶，并且可能预留一组溢出桶，以便溢出的时候快速分配。

```go
if h.B != 0 {
	var nextOverflow *bmap
	h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
	if nextOverflow != nil {
		h.extra = new(mapextra)
		h.extra.nextOverflow = nextOverflow
	}
}
```

（5）重点关注下 `makeBucketArray()` 方法，首先 `bucketShift()` 就是计算 2^B，也就是得到桶数组长度。当 b >= 4 的时候会提前分配出一些溢出桶备用。  
**因为当 b < 4 的时候，基本上不会溢出，根据平均负载阈值是 6.5 ，假设有 K 个桶，每个桶的平均容纳 6.5 个 kv 的时候，就需要增大 B 扩容，而当 B = 1 ～ 3 的时候，桶数量对应 1 ～ 8，扩容阈值是 6 ～ 52。换言之，桶基本上不会达到溢出的地步，map 就会触发增量扩容**

```go

func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// 这里为什么会有这么一个判断？
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.Bucket.Size_ * nbuckets
		up := roundupsize(sz, !t.Bucket.Pointers())
		if up != sz {
			nbuckets = up / t.Bucket.Size_
		}
	}
	// 分配桶数组
	if dirtyalloc == nil {
		buckets = newarray(t.Bucket, int(nbuckets))
	} else {
		buckets = dirtyalloc
		size := t.Bucket.Size_ * nbuckets
		if t.Bucket.Pointers() {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
	// 预分配一些溢出桶
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

#### 2.2.2 查询

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030230547252.png)

---

查询 map 实际上走的是 `runtime.mapaccess1()` 或者 `runtime.mapaccess2()`方法。我们不妨也来追踪一下。

```go
func main() {
	cache := make(map[string]int, 16)
	cache["A"] = 123

	_ = cache["A"]
}
```

如果用直接 key 去获得 v 走的是 `runtime.mapaccess1_faststr()` 方法。这是什么？其实是这样，我们可以在 `src/runtime/` 下发现有这么几个文件 `map_fast32_noswiss.go` 、`map_fast64_noswiss.go` 、`map_faststr_noswiss.go` 以及 `map_noswiss.go` 我们本次讲解主要围绕 `map_noswiss.go` 这个文件展开，其他的实际上是针对不同的 Key 的类型，做了特定的优化，其本质思想是类似的，所以这里不展开。  
其次，我们还有另外一种方式访问 map 就是 `v, exist := cache["A"]` ，当使用这种方法的时候，会返回 key 是否在 map 中的标记，底层走的是 `mapaccess2()` 方法，其核心与 `mapaccess1()` 类似，这里也不再赘述。

```shell
❯ dlv debug main.go
Type 'help' for list of commands.
(dlv) break runtime.mapaccess1_faststr
Breakpoint 1 set at 0x1045fcf90 for runtime.mapaccess1_faststr() /usr/local/go/src/runtime/map_faststr.go:13
(dlv) continue
> [Breakpoint 1] runtime.mapaccess1_faststr() /usr/local/go/src/runtime/map_faststr.go:13 (hits goroutine(1):1 total:1) (PC: 0x1045fcf90)
Warning: debugging optimized function
     8:         "internal/abi"
     9:         "internal/goarch"
    10:         "unsafe"
    11: )
    12:
=>  13: func mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer {
    14:         if raceenabled && h != nil {
    15:                 callerpc := getcallerpc()
    16:                 racereadpc(unsafe.Pointer(h), callerpc, abi.FuncPCABIInternal(mapaccess1_faststr))
    17:         }
    18:         if h == nil || h.count == 0 {
(dlv) args
t = ("*internal/abi.MapType")(0x10466bac0)
h = (*runtime.hmap)(0x14000052728)
ky = "A"
~r0 = (unreadable empty OP stack)
```

我们走进具体的方法去观察到底发生了什么？

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ...
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map read and map write")
	}
	hash := t.Hasher(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			if t.Key.Equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

（1）当 map 为 nil 或者无元素时，返回对应 val 的零值

```go
if h == nil || h.count == 0 {
	if err := mapKeyError(t, key); err != nil {
		panic(err) // see issue 23734
	}
	return unsafe.Pointer(&zeroVal[0])
}
```

（2）判断当前 map 是否正在被写入，如果发现有其他协程在写入 map，则直接 fatal。

```go
// 是否正在写入
cosnt hashWriting  = 4 // a goroutine is writing to the map
// 状态按位 & 来判断是否在写入，利用位运算是底层代码常见的方案
if h.flags&hashWriting != 0 {
	fatal("concurrent map read and map write")
}
```

（3）计算哈希、定位桶。通过 `t.Hasher()` 计算出 key 对应的 hash 值，并且计算出桶数组长度 - 1（用于计算 Hash 低 B 位的掩码），进而计算出数组中桶的位置。

```go
hash := t.Hasher(key, uintptr(h.hash0))
m := bucketMask(h.B)
// add 是地址 + 偏移量
// (hash&m) ==> hash % (1 << B)
b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
```

（4）扩容时 oldbuckets 指向旧表；未搬迁的桶优先从旧桶查。增量扩容时，旧表桶数是新表的一半，所以再右移一位 m，这样才能得到增量扩容前旧桶中对应的桶址。如果旧桶此时还没搬迁完，则将寻找目标改为旧桶

```go
if c := h.oldbuckets; c != nil {
	if !h.sameSizeGrow() {m >>= 1}
	oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
	if !evacuated(oldb) {
		// 转到旧桶搜索
		b = oldb
	}
}
// 获取 hash 值的高 8 位，如果高 8 位算下小于 5，则会 + 5，来避开下面提到的特殊标记。
top := tophash(hash)
```

b.tophash[i] 平时存放键哈希的高 8 位。但 **小于 minTopHash 的值都不是正常键**，而是状态标记。**（当 h ∈ {2, 3, 4} 的时候，表示该桶以及完成了搬迁）**

```go
const empty = 0    // 该槽为空
const emptyOne = 1    // 空，且其后全为空（搜索可早停）
const evacuatedX = 2    // 桶已搬到低位桶（当增量扩容时，新表长度是旧表两倍，所以会被拆分成两个桶）
const evacuatedY = 3    // 桶已搬到高位桶
const evacuatedEmpty = 4     // 已搬迁且为空
const minTopHash = 5    // 正常键的 tophash 值从 5 起

func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > emptyOne && h < minTopHash
}

func tophash(hash uintptr) uint8 {
	top := uint8(hash >> (goarch.PtrSize*8 - 8))
	if top < minTopHash {
		// 避开标记位
		top += minTopHash
	}
	return top
}
```

（5）遍历启动！先看外层循环，外层就是沿着溢出桶链表，遍历每个溢出桶。内层逻辑是首先一个桶最多放 8 个 kv 对，接着先用高 8 位 hash 判断，只有高 8 位 hash 相等再判断整个 key，进一步提升效率，不然每次都需要对比完整的 key。通过 KeySize 计算得到 k。接着校验 k 和查找的 key 是否相同，相同则返回 val。如果找遍了 overflowBmap 都没有找到 key，则直接返回 value 零值。

```go
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		// abi.OldMapBucketCount = 8
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 检查 key 是否相等，相等直接返回
			if t.Key.Equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				if t.IndirectElem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
```

#### 2.2.3 插入

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030234027743.png)

---

接着就是插入元素啦，让我们来康康走的是那个方法？

```go
func main() {
	// 这里用 interface{} 是因为一些基础类型做 key 的时候，都会走优化过的方法
	cache := make(map[interface{}]int, 16)
	cache[1] = 123

	println(cache[1])
}
```

可以看出，代码最后进入了 `runtime.mapassign()` 方法，这也正是 map 对应的插入相关的方法。
这里与读同理，对于一些基础类型 int、string 做 key 的 map，其底层有其他对应的优化方法，但本质类似，所以这里我们直接看本源方法 `runtime.mapassign()` 。

```shell
❯ dlv debug main.go
Type 'help' for list of commands.
(dlv) break runtime.mapassign
Breakpoint 1 set at 0x102f3997c for runtime.mapassign() /usr/local/go/src/runtime/map.go:615
(dlv) continue
> [Breakpoint 1] runtime.mapassign() /usr/local/go/src/runtime/map.go:615 (hits goroutine(1):1 total:1) (PC: 0x102f3997c)
Warning: debugging optimized function
   610: //
   611: // Do not remove or change the type signature.
   612: // See go.dev/issue/67401.
   613: //
   614: //go:linkname mapassign
=> 615: func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
   616:         if h == nil {
   617:                 panic(plainError("assignment to entry in nil map"))
   618:         }
   619:         if raceenabled {
   620:                 callerpc := getcallerpc()
(dlv) args
t = ("*internal/abi.MapType")(0x102f57ba0)
h = (*runtime.hmap)(0x14000052718)
key = unsafe.Pointer(0x14000052748)
~r0 = (unreadable empty OP stack)
```

让我们一探之究竟，首先一个疑问，插入为什么只传入了插入的 key，而没有 value？  
**插入操作的方法不只是 mapassign()，实际上 mapassign() 只负责返回一个插入的值的地址，接着由其他方法插入，所以 mapassign() 之所以叫 assign 而不叫 update 或者 insert**

```go
//go:linkname mapassign
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ... 工具检查
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}
	hash := t.Hasher(key, uintptr(h.hash0))

	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.Bucket) // newarray(t.Bucket, 1)
	}

again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
					elem = add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			if t.IndirectKey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.Key.Equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.NeedKeyUpdate() {
				typedmemmove(t.Key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, abi.OldMapBucketCount*uintptr(t.KeySize))
	}

	if t.IndirectKey() {
		kmem := newobject(t.Key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.IndirectElem() {
		vmem := newobject(t.Elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.Key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

（1）并发写保护并且设置标记位，懒分配 buckets。这里就是一个简单的并发写保护，并且计算出 key 的 hash 值，然后将 hmap 的 flag 设置为正在写的状态（**这里要注意的时候，写状态要在成功计算出 hash 后在修改，因为 t.Hasher() 也有可能 panic**）。其次，**我们在开头分配 bucket 的时候，当 B = 0 时，采用的是懒分配，也就是插入的时候分配，而分配的时间就是现在**。

```go
if h.flags&hashWriting != 0 {
	fatal("concurrent map writes")
}
hash := t.Hasher(key, uintptr(h.hash0))

h.flags ^= hashWriting

if h.buckets == nil {
	h.buckets = newobject(t.Bucket) // newarray(t.Bucket, 1)
}
```

（2） 定位目标桶，如果当前 map 处在扩容的状态，则帮助命中的桶进行扩容。因为这里涉及到了扩容的部分，内容较多，详情可以看 2.2.5 小节

```go
bucket := hash & bucketMask(h.B)
if h.growing() { growWork(t, h, bucket) }
b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
top := tophash(hash)
```

（3）遍历记录 tophash 首次可写的位置，以及这个位置对应的 key 和 val 的地址。

```go
var inserti *uint8
var insertk unsafe.Pointer
var elem   unsafe.Pointer
bucketloop:
for {
	for i := 0; i < 8; i++ {
		// 先用 tophash 过滤
	    if b.tophash[i] != top {
		    if isEmpty(b.tophash[i]) && inserti == nil {
		        // 首次见到 empty/evacuated 槽，先占住这个槽，后续如果查到相同 key 则更新
		        inserti = &b.tophash[i]
		        insertk = keySlotAddr(b, i)
		        elem    = valSlotAddr(b, i)
		    }
		    // 后面全空，整条链早停
		    if b.tophash[i] == emptyRest {
		        break bucketloop
		    }
			continue
	    }
	    k := keySlotAddr(b, i)
	    if t.IndirectKey() { k = *(*unsafe.Pointer)(k) }
	    // 这里是判断是不是以及存在这个 key，如果存在这直接更新对应的 kv
	    if t.Key.Equal(key, k) {
		    if t.NeedKeyUpdate() { typedmemmove(t.Key, k, key) }
		    elem = valSlotAddr(b, i)
		    goto done
	    }
	}
	if ovf := b.overflow(t); ovf != nil { b = ovf; continue }
	break
}
```

（4）在上一步中，如果没能找到一样的 key，则代表这是一次插入操作，插入操作就有可能引发 Hash 劣化，此时就需要先判断是否需要扩容了。这里因为涉及到扩容，所以也在 2.2.5 小节展开。

```go
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
	hashGrow(t, h)
	goto again // Growing the table invalidates everything, so try again
}
```

（5）在溢出桶中没有找到空位，就挂 overflow 溢出，这一步就是创建新的溢出桶（**这里会优先使用一开始创建 map 的时候冗余创建的溢出桶，当冗余桶不够的时候，则会创建新的溢出桶**）

```go
if inserti == nil {
	newb := h.newoverflow(t, b)
	inserti = &newb.tophash[0]
	insertk = add(unsafe.Pointer(newb), dataOffset)
	elem = add(insertk, abi.OldMapBucketCount*uintptr(t.KeySize))
}
```

（6）为 kv 预插入操作（**注意这里不是直接插入，尤其是 value**），首先会先按照 key 和 val 分配对象，然后先把这个空对象塞到槽内，然后使用 `typedmemmove(t.Key, insertk, key)` 将调用方传入的 Key 拷贝到最终位置，而 value 是由上层的调用方写入的。

```go
// store new key/elem at insert position
if t.IndirectKey() {
	kmem := newobject(t.Key)
	*(*unsafe.Pointer)(insertk) = kmem
	insertk = kmem
}
if t.IndirectElem() {
	vmem := newobject(t.Elem)
	*(*unsafe.Pointer)(elem) = vmem
}
typedmemmove(t.Key, insertk, key)
*inserti = top
h.count++
```

（7）收尾向上层返回最终确定要插入的 value 的地址

```go
done:
	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.IndirectElem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
```

#### 2.2.4 删除

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030234416131.png)

---

通过类似的 debug 手段，不难看出删除实际上是调用了 `runtime.mapdelete()` 方法

```go
//go:linkname mapdelete
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	// ... 工具配置
	if h == nil || h.count == 0 {
		if err := mapKeyError(t, key); err != nil {
			panic(err) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		fatal("concurrent map writes")
	}

	hash := t.Hasher(key, uintptr(h.hash0))

	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
			k2 := k
			if t.IndirectKey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.Key.Equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
			if t.IndirectKey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.Key.Pointers() {
				memclrHasPointers(k, t.Key.Size_)
			}
			e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			if t.IndirectElem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.Elem.Pointers() {
				memclrHasPointers(e, t.Elem.Size_)
			} else {
				memclrNoHeapPointers(e, t.Elem.Size_)
			}
			b.tophash[i] = emptyOne
			if i == abi.OldMapBucketCount-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = abi.OldMapBucketCount - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			if h.count == 0 {
				h.hash0 = uint32(rand())
			}
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		fatal("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

（1）安全性校验以及设置标记位

```go
if h == nil || h.count == 0 {
	if err := mapKeyError(t, key); err != nil {
		panic(err) // see issue 23734
	}
	return
}
if h.flags&hashWriting != 0 {
	fatal("concurrent map writes")
}

hash := t.Hasher(key, uintptr(h.hash0))
// 这里类似插入，标记位设置要在 Hasher 之后，因为 Hasher 方法可能 panic
h.flags ^= hashWriting
```

（2）定位目标桶，如果当前 map 正处于扩容状态，则帮助桶进行扩容，接着算出当前要查找的桶是哪个，然后算出高 8 位 hash 值用来做快速对比。

```go
bucket := hash & bucketMask(h.B)
if h.growing() {
	growWork(t, h, bucket)
}
b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
bOrig := b
top := tophash(hash)
```

（3）接着是和前面很类似的查找定位 key 的操作，一样的先通过 tophash 快筛，然后再对比 key

```go
for ; b != nil; b = b.overflow(t) {
	for i := uintptr(0); i < abi.OldMapBucketCount; i++ {
		if b.tophash[i] != top {
			if b.tophash[i] == emptyRest {
				break search
			}
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
		k2 := k
		if t.IndirectKey() {
			k2 = *((*unsafe.Pointer)(k2))
		}
		if !t.Key.Equal(key, k2) {
			continue
		}
	// ...
```

（4）当找到了对应的 key，则清理对应 bmap 的 tophash，以及之后的 kv 数据，这里就是计位置，然后删除 kv，然后将对应的 tophash 槽置为空。

```go
// Only clear key if there are pointers in it.
			if t.IndirectKey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.Key.Pointers() {
				memclrHasPointers(k, t.Key.Size_)
			}
			e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
			if t.IndirectElem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.Elem.Pointers() {
				memclrHasPointers(e, t.Elem.Size_)
			} else {
				memclrNoHeapPointers(e, t.Elem.Size_)
			}
			b.tophash[i] = emptyOne
```

（5）早停设计，记不记得我们之前在讲 tophash 的时候他有几个独特的标记位，其中有一个是早停的标记位，也就是当搜索的时候，如果发现当前` tophash == emptyRest` ，代表这个位置之后都没有 key 了，可以早停。**当我们删除 key 后，需要判断一下当前位能不能设置为早停位，如果可以，就可以往前寻找找到最早的可以设置为早停的地方。**

```go
if i == abi.OldMapBucketCount-1 {
	if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
		goto notLast
	}
} else {
	if b.tophash[i+1] != emptyRest {
		goto notLast
	}
}
// 代码能执行到这里就说明这个 key 后的所有槽都是空的
// 此时会将这个槽的 tophash 设置为早停标记
for {
	b.tophash[i] = emptyRest
	if i == 0 {
		if b == bOrig {
			break // beginning of initial bucket, we're done.
		}
	// 如果当前 key 是桶内第一个 key，则找到当前桶的前一个桶
		c := b
		for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {}
		i = abi.OldMapBucketCount - 1
	} else {
		i--
	}
	// 如果槽不为空，则直接退出
	if b.tophash[i] != emptyOne {
		break
	}
}
```

（6）计数器减一，如果此时没有 kv 在 map 中，重新设置 hash 种子

```go
notLast:
	h.count--
	if h.count == 0 {
		h.hash0 = uint32(rand())
	}
	break search
```

（7）退出，安全性检查以及标记位恢复

```go
// 校验是否有并发写
if h.flags&hashWriting == 0 {
	fatal("concurrent map writes")
}
h.flags &^= hashWriting
```

#### 2.2.5 扩容

##### 增量扩容

![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030235441064.png)
如上图，当存储密度很高且链足够长的时候，此时会触发增量扩容，也就是扩大 Hash 寻址的范围。数值上来看就是 Buket 数组翻倍，B + 1。进而把同一个溢出桶链上的 kv 打散。如下图
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251030235932898.png)

##### 等量扩容

我们的 map 会经常增删，但是删除的时候，并不会对桶进行回收，所以有可能导致链很长，但是存储密度很低，进而导致 hash 劣化。如下图这种极端情况，一个 buckt 里虽然只有两个 kv 对，但是要搜索完好几个桶才能定位。
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251031000400469.png)
此时会触发等量扩容，桶数组长度不变，转而压缩桶内数据，从而提高存储密度。如下图
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251031000433992.png)

---

通过上面的增删改查的代码，可以发现只有 map 插入元素的时候，可能会触发扩容。对应的代码片段也就是 `runtime.mapassign()` 部分。
当当前 map 不在扩容过程中，且满足下面两个条件之一的时候，则会开启扩容：

1. `overLoadFactor(h.count+1, h.B)` 判断如果再插入一个元素，会不会导致存储密度超出阈值，进而导致 hash 劣化
2. `tooManyOverflowBuckets(h.noverflow, h.B)` 判断是否是存储密度太小，但是溢出桶太多导致的 hash 劣化
   上面两点也就对应了我们上面所说到的两种 map 的扩容方式。

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ...
	// Did not find mapping for key. Allocate new cell & add entry.
	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	// 扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	// ...
}
```

触发扩容后，会进入 `hashGrow()` 方法，这里会执行 map 扩容的流程。

```go
func hashGrow(t *maptype, h *hmap) {
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}
}
```

（1）首先，无论是等量扩容还是增量扩容，都会创建新的 bucket 数组，只不过等量不会扩大这个数组的长度而已。这里判断一下要分配的桶数组长度，增量扩容的话，数组长度翻倍。

```go
if !overLoadFactor(h.count+1, h.B) {
	bigger = 0
	h.flags |= sameSizeGrow
}
oldbuckets := h.buckets
newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
```

（2）设置标记位，修改 map 头中的元数据，更新 B、flag，将头中的新老桶指针分别指向旧桶和新桶数组，将扩容进度 nevacuate 置为 0，新桶的溢出桶数量置为 0。

```go
flags := h.flags &^ (iterator | oldIterator)
if h.flags&iterator != 0 {
	flags |= oldIterator
}
// commit the grow (atomic wrt gc)
h.B += bigger
h.flags = flags
h.oldbuckets = oldbuckets
h.buckets = newbuckets
h.nevacuate = 0
h.noverflow = 0
```

（3）将 map 头中的 extra 的旧溢出桶数组字段 `hmap.extra.oldoverflow` 指向溢出桶字段，因为渐进式扩容，需要一步一步搬溢出桶中的数据，然后溢出桶数组改为 `hmap.extra.overflow` 置为空，用来存储新的 bucket 的溢出桶地址（**前面提到过，不只有 bmap 中虽然会逻辑层面划分后面的部分给 kv 数组以及 nextOverflow 指针，但是为了防止 GC 误回收那片区域，所以这里 extra 中的字段会额外存储对应的 nextOverflow 指针，这里实际上也有点意思，感兴趣的读者可以了解一下为什么要单独用 h.extra 存放所有的溢出桶，而 k 和 v 不需要**）。最后就是将一开始创建好待用的溢出桶确保其存在就行。

```go
if h.extra != nil && h.extra.overflow != nil {
	// Promote current overflow buckets to the old generation.
	if h.extra.oldoverflow != nil {
		throw("oldoverflow is not nil")
	}
	h.extra.oldoverflow = h.extra.overflow
	h.extra.overflow = nil
}
if nextOverflow != nil {
	if h.extra == nil {
		h.extra = new(mapextra)
	}
	h.extra.nextOverflow = nextOverflow
}
```

至此，扩容就完毕了，接下来就是迁移。从之前的插入源码可以得到，迁移部分的代码入口是`runtime.growWord()` 方法，这里会先帮助本次插入涉及到的桶完成迁移，然后再额外辅助迁移一个桶。

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 确保本次命中的桶完成迁移
	evacuate(t, h, bucket&h.oldbucketmask())

	// 多帮忙搬迁一个桶推进增量扩容的进度
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

这里我们也看到了，实际上搬迁的主要方法是 `exacuate()` 方法，进一步剖析该方法（注释形式）

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	// 获取到桶的位置
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
	newbit := h.noldbuckets()
	// 检查旧桶是否迁移完毕，当未完成迁移，则迁移
	if !evacuated(b) {
		// 这里会计算两个 rehash 的目标位置，第二个位置会根据扩容的类型来选择
		// 我们知道等量扩容时其 hash 地址是不会变的，这些地址不变的 kv 这里就对应了地址 x
		// 当增量扩容模式下，有的 kv 会被 rehash 到其他溢出桶，这里对应的是下面的 y
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.BucketSize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, abi.OldMapBucketCount*uintptr(t.KeySize))

		if !h.sameSizeGrow() {
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets,(oldbucket+newbit)*uintptr(t.BucketSize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, abi.OldMapBucketCount*uintptr(t.KeySize))
		}
		// 这里的遍历思路也是外层沿着溢出桶指针，内层遍历溢出桶内的 8 个 kv
		for ; b != nil; b = b.overflow(t) {
			// 当前桶 keys[0] 起始地址
			k := add(unsafe.Pointer(b), dataOffset)
			// 当前桶 values[0] 起始地址
			e := add(k, abi.OldMapBucketCount*uintptr(t.KeySize))
			for i := 0; i < abi.OldMapBucketCount; i, k, e = i+1, add(k, uintptr(t.KeySize)), add(e, uintptr(t.ValueSize)) {
				top := b.tophash[i]
				// 高八位快速判断当前槽内有没有 kv
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				// 无异常情况，获取到需要搬迁的 key
				k2 := k
				if t.IndirectKey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				// 这里是判断这个 key 的目的地，只有增量扩容的时候才会用到 y
				var useY uint8
				if !h.sameSizeGrow() {
					// 如果是增量扩容，先计算 hash 值
					hash := t.Hasher(k2, uintptr(h.hash0))

					if h.flags&iterator != 0 && !t.ReflexiveKey() && !t.Key.Equal(k2, k2) {
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == abi.OldMapBucketCount {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, abi.OldMapBucketCount*uintptr(t.KeySize))
				}
				dst.b.tophash[dst.i&(abi.OldMapBucketCount-1)] = top
				if t.IndirectKey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.Key, dst.k, k) // copy elem
				}
				if t.IndirectElem() {
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
					typedmemmove(t.Elem, dst.e, e)
				}
				dst.i++
				dst.k = add(dst.k, uintptr(t.KeySize))
				dst.e = add(dst.e, uintptr(t.ValueSize))
			}
		}

		if h.flags&oldIterator == 0 && t.Bucket.Pointers() {
			b := add(h.oldbuckets, oldbucket*uintptr(t.BucketSize))
			ptr := add(b, dataOffset)
			n := uintptr(t.BucketSize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
```

（1）定位迁移的目的地，这里需要通过判断是否是增量扩容来进一步判断是否需要另一个地址（**因为增量扩容有的 key 会被 rehash 到新桶数组的不同位置**）

```go
b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
newbit := h.noldbuckets() // = 1 << (B_old)
if !evacuated(b) {
    var xy [2]evacDst
    x := &xy[0]
    x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.BucketSize))) // 低位桶 X（new[i]）
    x.k = add(unsafe.Pointer(x.b), dataOffset)
    x.e = add(x.k, abi.OldMapBucketCount*uintptr(t.KeySize))

    if !h.sameSizeGrow() { // 只有“增量扩容”才有 Y 桶
        y := &xy[1]
        y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.BucketSize))) // 高位桶 Y（new[i+newbit]）
        y.k = add(unsafe.Pointer(y.b), dataOffset)
        y.e = add(y.k, abi.OldMapBucketCount*uintptr(t.KeySize))
    }
    ...
}
```

（2）首先沿着 overflow 链扫描 kv，然后用 key 的 hash 值的地位是 0/1 判断其在新桶中是原位置 x 还是新位置 y，接着将 kv 以及 tophash 写入目的地。

```go
for ; b != nil; b = b.overflow(t) {
    k := add(unsafe.Pointer(b), dataOffset)
    e := add(k, abi.OldMapBucketCount*uintptr(t.KeySize))
    for i := 0; i < abi.OldMapBucketCount; i, k, e = i+1, add(k, KSize), add(e, VSize) {
        top := b.tophash[i]
        if isEmpty(top) { b.tophash[i] = evacuatedEmpty; continue }
        if top < minTopHash { throw("bad map state") }

        // 取 key 地址（考虑间接存放）
        k2 := k
        if t.IndirectKey() { k2 = *(*unsafe.Pointer)(k2) }

        // 选 X 还是 Y（只有增量扩容才分流）
        var useY uint8
        if !h.sameSizeGrow() {
            hash := t.Hasher(k2, uintptr(h.hash0))
            if h.flags&iterator != 0 && !t.ReflexiveKey() && !t.Key.Equal(k2,k2) {
                // 非自反键（NaN!=NaN），用旧 tophash 低位稳定分流
                useY = top & 1
                top = tophash(hash) // 同时重算新 tophash
            } else {
                if hash & newbit != 0 { useY = 1 } // 高位为 1 则去 Y
            }
        }

        // 标记该槽已搬迁到 X/Y
        b.tophash[i] = evacuatedX + useY // evacuatedX+1 == evacuatedY
        dst := &xy[useY]

        // 目标桶位已满则挂溢出并重置写指针
        if dst.i == abi.OldMapBucketCount {
            dst.b = h.newoverflow(t, dst.b)
            dst.i = 0
            dst.k = add(unsafe.Pointer(dst.b), dataOffset)
            dst.e = add(dst.k, abi.OldMapBucketCount*uintptr(t.KeySize))
        }

        // 写入目标桶的 tophash / key / value
        dst.b.tophash[dst.i&(abi.OldMapBucketCount-1)] = top
        if t.IndirectKey() { *(*unsafe.Pointer)(dst.k) = k2 } else { typedmemmove(t.Key, dst.k, k) }
        if t.IndirectElem(){ *(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e) } else { typedmemmove(t.Elem, dst.e, e) }
        dst.i++
        dst.k = add(dst.k, uintptr(t.KeySize))
        dst.e = add(dst.e, uintptr(t.ValueSize))
    }
}
```

（3） 辅助迁移收尾，推进标识位的更新

```go
if h.flags&oldIterator == 0 && t.Bucket.Pointers() {
    b := add(h.oldbuckets, oldbucket*uintptr(t.BucketSize))
    ptr := add(b, dataOffset)
    n := uintptr(t.BucketSize) - dataOffset
    memclrHasPointers(ptr, n)
}

if oldbucket == h.nevacuate {
    advanceEvacuationMark(h, t, newbit)
}
```

#### 2.2.6 遍历

我们先来看 go-map 中一个经典的问题，关注下面的代码，请问输出是什么？

```go
func main() {
	cache := make(map[int]string, 16)

	cache[1] = "A"
	cache[2] = "B"
	cache[3] = "C"
	cache[4] = "D"


	for k, v := range cache {
		println(k, " + ", v)
	}
}
```

**答曰：** 不知道～

```go
❯ go build -o main main.go
❯ ./main
4  +  D
1  +  A
3  +  C
2  +  B
❯ ./main
2  +  B
4  +  D
3  +  C
1  +  A
```

我们发现同一段代码，遍历顺序却随机，答案就藏在 map 的迭代器中，**go 团队设计迭代 map 的时候故意设计成了随机的，就是为了消除程序对有序遍历 map 的依赖，这主要是对可移植性和安全性的考量。如果程序依赖固定的顺序迭代 map，那这段代码可能在不同的硬件平台上的执行结果不同，导致代码在不同平台上无法运行**。具体可以看下面链接的一些内容。  
https://stackoverflow.com/questions/55925822/why-are-iterations-over-maps-random  
如下图，迭代器会随机定位到某个 Bucket 的某个溢出桶的某个位置开始迭代，从而知道随机性。
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251031000913089.png)

---

接着我们来看看迭代部分的代码，通过类似的定位手段，可以找到其底层是基于 `runtime.mapiterinit()` 初始化一个迭代器，然后使用 `runtime.mapiternext()` 方法进行迭代的，我们展开来看看，首先来看看初始化迭代器。  
首先先介绍一下 hmap 的迭代器的结构 hiter

```go
type hiter struct {
	// 当前迭代的 key 的地址
	key         unsafe.Pointer
	// 当前迭代的 val 的地址
	elem        unsafe.Pointer
	// 当前 map 的元信息
	t           *maptype
	// hmap 结构体指针
	h           *hmap
	// 桶数组快照
	buckets     unsafe.Pointer
	// 当前所在的溢出桶（bmap）
	bptr        *bmap
	// 新老桶数组对应的溢出桶；
	overflow    *[]*bmap
	oldoverflow *[]*bmap
	// 起始桶索引
	startBucket uintptr
	// 桶内起始偏移
	offset      uint8
	// 是否已经 从桶数组尾部绕回 到开头
	wrapped     bool
	// 迭代开始时的 B 值
	B           uint8
	// 当前所处的桶内的索引
	i           uint8
	// 当前所处的桶（bucket）索引，这个要和上面的 bptr 做区分
	bucket      uintptr
	// 用于在 扩容/渐进搬迁 时校验：当 oldbuckets 尚存在，读路径/迭代需要按规则优先从旧桶读未搬迁元素；
	// checkBucket 协助判断该看旧还是新、或是否跳过已 evacuated 的桶。
	checkBucket uintptr
	clearSeq    uint64
}
```

```go
//go:linkname mapiterinit
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	// ...

	// 如果 map 为空，直接返回
	it.t = t
	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{}) != 8+12*goarch.PtrSize {
		throw("hash_iter size incorrect")
	}
	// 初始化 iter 中的一些值
	it.h = h
	it.clearSeq = h.clearSeq

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if !t.Bucket.Pointers() {
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// 决定 it 从哪里出发，这里明显可以看到使用了一个随机的初始点
	// 这里定随机桶和随机桶内的偏移量
	r := uintptr(rand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (abi.OldMapBucketCount - 1))

	// iterator state
	it.bucket = it.startBucket

	// 这里会设置 hmap，告知当前运行时存在迭代器
	// 其他迁移/删除逻辑会参考这个标记做进一步安全性的检查，防止出错
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}
	// 执行迭代流程
	mapiternext(it)
}
```

（1）空值和安全性校验

```go
it.t = t
if h == nil || h.count == 0 {
	return
}

if unsafe.Sizeof(hiter{}) != 8+12*goarch.PtrSize {
	throw("hash_iter size incorrect")
}
```

（2）初始化 iter 中的一些值

```go
it.h = h
it.clearSeq = h.clearSeq
// grab snapshot of bucket state
it.B = h.B
it.buckets = h.buckets
if !t.Bucket.Pointers() {
	h.createOverflow()
	it.overflow = h.extra.overflow
	it.oldoverflow = h.extra.oldoverflow
}
```

（3）这里不然看出，决定迭代的起始桶和桶内偏移是用 `rand()` 选择的，这也就对应了一开始的部分

```go
// 决定 it 从哪里出发，这里明显可以看到使用了一个随机的初始点
// 这里定随机桶和随机桶内的偏移量
r := uintptr(rand())
it.startBucket = r & bucketMask(h.B)
it.offset = uint8(r >> h.B & (abi.OldMapBucketCount - 1))
// iterator state
it.bucket = it.startBucket

// 这里会设置 hmap，告知当前运行时存在迭代器
// 其他迁移/删除逻辑会参考这个标记做进一步安全性的检查，防止出错
if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
	atomic.Or8(&h.flags, iterator|oldIterator)
}
// 执行迭代流程
mapiternext(it)
```

接下来进入迭代流程

```go
//go:linkname mapiternext
func mapiternext(it *hiter) {
	h := it.h
	// ...
	// 基础并发检查
	if h.flags&hashWriting != 0 {
		fatal("concurrent map iteration and map write")
	}
	// 分别获取 map 元数据，桶数组，当前的溢出桶，桶内下标等
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
		// 如果当前处于开始遍历的位置，并且已经遍历完一轮，则结束迭代
		if bucket == it.startBucket && it.wrapped {
			it.key = nil
			it.elem = nil
			return
		}
		// 如果当前处于扩容状态，则将桶设置为旧桶
		if h.growing() && it.B == h.B {
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
			if !evacuated(b) {
				// 如果旧桶还没有迁移结束，则需要设置检查桶
				checkBucket = bucket
			} else {
				// 旧桶迁移未结束，则不需要设置检查桶，并且转去搜索新桶
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
				checkBucket = noCheck
			}
		} else {
			// 不处于扩容状态，则无需设置检查桶
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
			checkBucket = noCheck
		}
		bucket++
		// 如果桶下标到头了，则设置回到开头，并设置遍历轮回的标记
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	// 遍历桶内的槽位
	for ; i < abi.OldMapBucketCount; i++ {
		offi := (i + it.offset) & (abi.OldMapBucketCount - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// 空槽直接跳过
			continue
		}
		// 获取到槽内的 key
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.KeySize))
		if t.IndirectKey() {
			k = *((*unsafe.Pointer)(k))
		}
		// 获取到槽内的 val
		e := add(unsafe.Pointer(b), dataOffset+abi.OldMapBucketCount*uintptr(t.KeySize)+uintptr(offi)*uintptr(t.ValueSize))
		// 如果设置了检查旧桶，并且是增量扩容，此时需要判断这个 key 处于新桶中的原位置还是被放在了新桶中的新位置
		// （rehash 的时候用高位 0/1 来判断要送去新桶还是旧桶）
		if checkBucket != noCheck && !h.sameSizeGrow() {
			if t.ReflexiveKey() || t.Key.Equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := t.Hasher(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					// 如果这个 key 处于新桶，则跳过，检查下一个 key，等检查新桶的时候在输出
					continue
				}
			} else {
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if it.clearSeq == h.clearSeq &&
			((b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
				!(t.ReflexiveKey() || t.Key.Equal(k, k))) {
			// 如果当前 key 经过检查可以确定是可信的，那直接用这个数据
			it.key = k
			if t.IndirectElem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// 如果 map 在迭代期间发生了一些增删改等，则调用方法获取对应的值
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		// 更新迭代器进度
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

（1）如果当前在起始点，则判断当前是不是以及循环一轮了，如果是，则代表迭代结束，直接返回

```go
if bucket == it.startBucket && it.wrapped {
	it.key = nil
	it.elem = nil
	return
}
```

（2）确定要遍历的桶数组，如果当前正处于增量扩容阶段，则需要去旧桶中检查，否则直接在新桶中检查就行。这里有两个比较关键的点：

1. 其实这里比较关键的代码是 `oldbucket := bucket & it.h.oldbucketmask()`，**这句代码的含义是，取 bucket 的低 B - 1 位，通过低 B 位能判断出这个桶的旧桶位置。举个例子，假设原来的 B 为 3（8 个桶），翻倍后 B 位 4（16 个桶）。新桶的第一个桶下标是 0，第九个桶下标是 8，这两个桶的数据按照扩容原则，应该都属于旧桶中的的 0 号桶，比如此时 0 号桶低 3 位（B - 1）是 000，8 的低 3 位也是 000，这样就能确定他们都属于 0 号桶。**
2. bucket 字段始终指向新桶，而 b 字段有可能指向新桶也有可能指向旧桶。

```go
if h.growing() && it.B == h.B {
	oldbucket := bucket & it.h.oldbucketmask()
	b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
	if !evacuated(b) {
		// 如果旧桶还没有迁移结束，则需要设置检查桶
		checkBucket = bucket
	} else {
		// 旧桶迁移未结束，则不需要设置检查桶，并且转去搜索新桶
		b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
		checkBucket = noCheck
	}
} else {
	// 不处于扩容状态，则无需设置检查桶
	b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
	checkBucket = noCheck
}
```

（3）遍历桶内数据，获取到 kv

```go
for ; i < abi.OldMapBucketCount; i++ {
		offi := (i + it.offset) & (abi.OldMapBucketCount - 1)
		// ...
```

（4）如果在增量扩容的时候，因为桶数组长度翻倍，所以原来的 bucket 的一些数据可能会被 rehash 到新的 bucket 的溢出桶中，所以此时获取到 key 后，要检查其是否属于当前旧桶，如果这个 key 未来会被分流到新桶中，则跳过，等遍历到新桶的时候，再输出。

```go
if checkBucket != noCheck && !h.sameSizeGrow() {
	if t.ReflexiveKey() || t.Key.Equal(k, k) {
		hash := t.Hasher(k, uintptr(h.hash0))
		if hash&bucketMask(it.B) != checkBucket {
			// 如果这个 key 处于新桶，则跳过，检查下一个 key，等检查新桶的时候在输出
			continue
			}
		} else {
			if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
				continue
			}
		}
		}
	if it.clearSeq == h.clearSeq &&
		((b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
		!(t.ReflexiveKey() || t.Key.Equal(k, k))) {
		// 如果当前 key 经过检查可以确定是可信的，那直接用这个数据
		it.key = k
		if t.IndirectElem() {
			e = *((*unsafe.Pointer)(e))
		}
		it.elem = e
	} else {
		// 如果 map 在迭代期间发生了一些增删改等，则调用方法获取对应的值
		rk, re := mapaccessK(t, h, k)
		if rk == nil {
			continue // key has been deleted
		}
		it.key = rk
		it.elem = re
	}
```

（5）更新迭代器进度

```go
// 更新迭代器进度
	it.bucket = bucket
	if it.bptr != b {
		it.bptr = b
	}
	it.i = i + 1
	it.checkBucket = checkBucket
	return
}
b = b.overflow(t)
i = 0
goto next
```

## 彩蛋

为什么不用 `interface{}` 来表示泛型呢？这样代码可读性会提高不少。

1. 规范性角度考虑：`interface{}` 规范性上来讲是一种动态类型的容器（鸭子类型），是运行是多态，真正的泛型是编译时推断类型。所以用 `interface{}` 表达泛型本身就不规范。
2. 存储角度考量：我们来看看当使用 `interface{}` 来组成这个结构体，他们之间的差距会有多大？我们可以简单试探一下。

```go
package main

import (
	"unsafe"
)

type bmap struct {
	tophash [8]uint8
	key [8]interface{}
	val [8]interface{}
	nextBmap *bmap
}

type bmap1 struct {
	tophash [8]uint8
	key [8]uint8
	val [8]uint8
	nextBmap *bmap
}

func main() {
	var map1 bmap
	var map2 bmap1

	println(unsafe.Sizeof(map1))
	println(unsafe.Sizeof(map2))
}
```

输出如下

```shell
❯ go build -o main main.go
❯ ./main
272
32
```

在这种极端的情况下，二者所占据的内存差异极大，我想这是一个为什么不用 `interface{}` 的点。而且 go 底层对不同类型的 kv 对的 map 都做了特殊优化，用 `interface{}` 是又慢又重。
## 参考资料
https://pkg.go.dev/gitea.wit.com/jcarr/iter/abi#OldMapBucketCountBits  
https://draven.co/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E5%93%88%E5%B8%8C%E5%87%BD%E6%95%B0  
https://stackoverflow.com/questions/55925822/why-are-iterations-over-maps-random  
https://zhuanlan.zhihu.com/p/597483155  
https://www.cnblogs.com/baxianhua/p/11699068.html
