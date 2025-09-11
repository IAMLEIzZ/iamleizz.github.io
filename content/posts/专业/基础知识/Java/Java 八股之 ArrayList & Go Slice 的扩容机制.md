---
title: Java ArrayList & Go Slice 扩容
date: 2025-09-03T11:53:04+08:00
lastmod: 2025-09-03T11:53:04+08:00
draft: false
tags:
  - Java
  - Golang
  - 计算机基础
categories: 
author: 小石堆
---
	Java 中的 ArrayList 类似 Golang 里的 Slice，都是动态数组
## ArrayList
### 什么是 ArrayList
`ArrayList` 是 Java 中最常见的动态数组之一，他实现了 List 接口，底层基于数组实现，但能自动扩容，可以理解为变长数组的一种。
```Java
List<String> list = new ArrayList<>();
list.add("zhangsan");
list.add("lisi");
System.out.println(list);
```
### ArrayList 的扩容原理
1. 当创建一个 `ArrayList` 的时候，默认初始的容量为 10，其内部实现类似 `Object[] elementData = new Object[10]`
2. 当元素超过容量时，arraylist 就会触发扩容机制，其触发条件为 `if(size == elementData.length)`
3. 扩容策略如下 `newCapacity = oldCapacity + (oldCapacity + 1)`，简言之就是扩容为原来的 1.5 倍。
这里涉及到扩容过程的主要函数是 `grow()`
```Java
private void grow(int minCapacity) {
	int oldCapacity = elementData.length;
	int newCapacity = oldCapacity + (oldCapacity + 1);

	if(newCapacity - minCapacity < 0) {
		newCapacity = minCapacity;
	}

	// 创建新数组，复制旧数据
	elementData = Arrays.copyOf(elementData, newCapacity)
}
```
这里的扩容规则实际上并没有 Go 的复杂，但是要理解 `minCapacity`
### minCapacity
`minCapacity` 指的是 “最小所需容量”，即为了成功执行当前操作，`ArrayList` 底层的 Object 数组至少具备的容量，所以，这个参数是由 Action 决定的。
比如，一般我们操作 `ArrayList` 的时候，有 `add(E e)` 和 `addAll(Collection c)` ，那么 `minCapacity` 就由这两个函数来决定。
#### 场景一：无参构造 ArrayList 后，添加第一个元素
```Java
List<String> list = new ArrayList<>();
list.add("A");
```
**步骤分析：**
1. 调用 `add("A")`，其内部调用 `ensureCapacityInternal(size + 1)`
2. 此时 size = 0
3. 所以 `minCapacity = size + 1 = 1`
4. 最终 `minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity)`
5. 因为当前数组容量 0 < 10，所以触发扩容，直接将数组扩容为 10
#### 场景二：数组已满，继续添加单个元素
```Java
list.add("B");
```
**步骤分析：**
1. 调用 `add("B")`，其内部调用 `ensureCapacityInternal(size + 1)`
2. 此时 size  = 10
3. 所以 `minCapacity = size + 1 = 11`
4. 10 < 11，所以需要扩容
5. `grow()`扩容，`newCapacity = 10 + (10 >> 1) = 15`
6. 15 > 11，所以数组最后容量为 15
#### 场景三：批量添加元素（addAll）
```Java
List<String> list = new ArrayList<>(5);
list.add("A");
list.add("B");
list.add("C");

List<String> anotherList = Arrays.asList("X", "Y", "Z", "W");

list.addAll(anotherList);
```
**步骤分析：**
1. 调用 `addAll`，内部算出 `minCapacity = 3 + 4  = 7`
2. 检查当前容量 5 < 7，触发扩容
3. `grow()`扩容，`newCapacity =  5 + (5 >> 1) = 7`
4. 7 == 7，所以刚好扩容到 7
### 总结
1. `ArrayList` 扩容会先根据添加元素的数量，计算出一个满足当前 add 需求的一个值
2. 如果这个值不超过当前 `ArrayList` 的容量，则不会扩容，如果扩容，则会进入 `grow()`
3. 这个方法内会将原本容量 * 1.5，然后观察够不够，如果不够，则直接设置为所需要的容量
4. 最后使用 `Arrays.copyOf` 迁移
## Slice
### 什么是Slice
`Slice` 是 Go 中的动态数组，具备动态扩容的能力
```Go
s1 := make([]int)
s3 := make([]int, x)
s2 := make([]int, x, y)
```
### Slice 结构
```Go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
`Slice` 底层实际上就是一个结构体，包含了长度和容量，以及实际存储数据的位置
### Slice 扩容原理
`Slice` 的扩容也比较简单
1. 如果需要的容量 > 原来容量 * 2 ==> 直接分配需要的容量
2. 如果不需要 < 原来容量 * 2，则先检查当前容量是否 < 256，小于直接翻倍
3. 如果当前容量 > 256，则进行这样的操作 `oldmem += (oldmem + 3 * 256) / 4`
**当分配好内存后，然后将久的内存中的元素复制过去**
也就是说，在分配内存后，`slice` 的内存地址会发生变化
### 扩展
#### 扩展 1
```Go
s := make([]int, 5, 10)
s1 := s
s = append(s, s[:]) 
s = append(s, 10) // 发生扩容
```
此时，s 发生扩容后，他 `slice_header` 中的 `array` 发生变化，于是不再和 s1 共享一块内存，次后 s 和 s1 的增删再无关联，s1 依然指向 s 不要的旧内存空间。
#### 扩展 2
分配内存这里还有一个坑，就是假设我现在分配一个 60 的容量，但是由于 Go 的内存管理和内存分配机制，内存是按照 `mspan` 分配的，`mspan` 是分等级的（每一级大小一般是 2 的 n 次方），所以会向上取整，直接分配一个 64 过去（减少碎片）。如下例：
```Go
func Test_slice(t *testing.T){    
	s := make([]int,512)      
	s = append(s,1)    
	t.Logf("len of s: %d, cap of s: %d",len(s),cap(s))
}
```
	答案：len: 513, cap: 848

	首先，由于切片 s 原有容量为 512，已经超过了阈值 256，因此对其进行扩容操作会采用的计算共识为 512 * (512 + 3 * 256) / 4 = 832

	其次，在真正申请内存空间时，我们会根据切片元素大小乘以容量计算出所需的总空间大小，得出所需的空间为 8byte * 832 = 6656 byte

	再进一步，结合分配内存的 mallocgc 流程，为了更好地进行内存空间对其，golang 允许产生一些有限的内部碎片，对拟申请空间的 object 进行大小补齐，最终 6656 byte 会被补齐到 6784 byte 的这一档次. 