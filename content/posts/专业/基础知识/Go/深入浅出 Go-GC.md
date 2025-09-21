---
title: 深入浅出 Go-GC
date: 2025-09-19T13:20:30+08:00
lastmod: 2025-09-19T13:20:30+08:00
draft: false
tags:
  - Golang
  - Java
  - 垃圾回收机制
categories:
author: 小石堆
---
	&&&
![image.png](http://43.139.219.135:9000/blog-pic/images/20250921173754411.png)

# Golang GC流程
## Go 如何启动 GC
Go 触发 GC 有三种情况：
1. 主动调用 runtime.gc() 可能会触发 Gc
2. 当分配对象时可能会触发 Gc
3. 守护协程定时Gc

	runtime/mgc.go
```Go
const (    
	// 根据堆分配内存情况，判断是否触发
	GC    gcTriggerHeap gcTriggerKind = iota    
	// 定时触发
	GC    gcTriggerTime    
	// 手动触发
	GC    gcTriggerCycle 
}	
```
```Go
func (t gcTrigger) test() bool {    
// ...    
	switch t.kind {    
	case gcTriggerHeap:        
		// ...        
		// 如果是堆内存分配导致的 GC，会 Check 当前堆内存的使用情况
		trigger, _ := gcController.trigger()        
		return atomic.Load64(&gcController.heapLive) >= trigger    
	case gcTriggerTime:        
		// 如果是守护协程 GC，则会check当前距离上一次 GC 是否已经达到 2 min
		if gcController.gcPercent.Load() < 0 {            
			return false        
		}        
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))        
		return lastgc != 0 && t.now-lastgc > forcegcperiod  
	case gcTriggerCycle:        
		// ...        
		// 如果是手动触发 GC，check 前一轮 GC 有没有结束
		return int32(t.n-work.cycles) > 0    
	}    
	return true
}
```
## Gc - Start
Golang的标记清扫算法的关键点就是标记这一步，其采用三色标记法，从 Root 对象开始进行可达性分析，关键步骤如下：
1. 获取 GC 锁，保证没有其他 GC 正在进行中
2. `StopTheWorldSema()`：STW 主要就是用来保证标记的前置工作的，因为标记过程中的 Root 对象一定是要确定好的
	这里深入源码可以了解到 **Go 的基于抢占 + 协作式**的调度，后期我会写一篇新的文章来表述
3. `gcMarkRootPrepare()` & `gcMarkTinyAllocs()` 这两步就是标记所谓的 Root 对象，可达性分析的初始：
	1. `gcMarkRootPrepare()` 主要是启动一些 Root 对象标记的工作线程，并不是直接进行标记，这些工作的线程数量和内存大小有关，例如：这里说的 Root 对象主要是.BSS段和.DATA段的对象，这些段大小，Go 底层会按照默认的块大小将他们分块，然后启动对应数量的携程到时候用于标记
	2. `gcMarkTinyAllocs()` 主要是标记 Tiny 对象，因为 GO 内存模型的缘故，极小对象并没有分配在堆上，所以为了防止可达性分析出错，因此直接标记了 Tiny 对象，视为 Root 对象
4. `startTheWorldWithSema()`，这里就是 stoptheworld 的逆操作，用于恢复所有协程
其实上述就是 gcStart 的内容，代表着 Gc 开始，而后，通过 G0 调度，所有的并发标记协程会和用户协程交替标记。
## G0 - schedule Gc 协程
因为并发标记，所以 Go 调度 GC 协程走的实际上是还是通用的协程调度算法 schedule() 方法，深入 schedule() 源码，我们知道当 G 在调度协程的时候，是由 G0 协程来完成的，当进入方法后，会进入一个教 findRunnable() 的方法，来找到已经就绪等待运行的 G，这里会涉及到 GMP 模型的 Steal 机制和 Handoff 机制。
```Go
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
	// ...
top:
	// ...
	// 尝试获取一个并发标记协程
	if gcBlackenEnabled != 0 {
		gp, tnow := gcController.findRunnableGCWorker(pp, now)
		// 如果能找到 GC 相关的G，则返回对应的 G
		if gp != nil {
			return gp, false, true
		}
		now = tnow
	}

		// 周期性检查全局队列，防止全局队列饥饿
		// 也就是说每 61 次，就会发生一次全局 G 的调用
	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget()
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
	
	// ...

	// 获取本地 P 队列中的 G
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

	// 获取全局队列中的 G
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp, q, qsize := globrunqgetbatch(int32(len(pp.runq)) / 2)
		unlock(&sched.lock)
		if gp != nil {
			if runqputbatch(pp, &q, qsize); !q.empty() {
				throw("Couldn't put Gs into empty local runq")
			}
			return gp, false, false
		}
	}

	// 优先轮询 I/O 是否有就绪 goroutine。
	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}

	// 从其他队列中获取 work stealing
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}

		now = tnow
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}

	// ....
	
	// 如果上面的 case 都没有执行，代表此时可以释放 P，则再进行一次 double check 后，释放
	lock(&sched.lock)
	if sched.gcwaiting.Load() || pp.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	// check 全局 G 中还有没有 G
	if sched.runqsize != 0 {
		gp, q, qsize := globrunqgetbatch(int32(len(pp.runq)) / 2)
		unlock(&sched.lock)
		if gp == nil {
			throw("global runq empty with non-zero runqsize")
		}
		if runqputbatch(pp, &q, qsize); !q.empty() {
			throw("Couldn't put Gs into empty local runq")
		}
		return gp, false, false
	}
	// 如果没有当前 M 会和 P work stealing，M 进入休眠（线程阻塞，等待唤醒而不是就绪哦）
	stopm()
	goto top
}
```
OK！不难看出这一版的 **G0 的调度机制**：
	GC -> 本地 P -> 全局 P -> 异步阻塞 IO 的 G ->  Stealing P -> GoPark (M  Park 阻塞)
当 G0 拉起了一个 GC worker，并发标记协程也就上号了。
## GC - Mark
当 GC 协程被调度后，会执行对应的标记方法，对应不同的模式，有不同的工作类型，这里主要是 `gcDrain()`方法。其分为 `gcDrainMarkWorkerDedicated()`、`gcDrainMarkWorkerFractional()` 以及`gcDrainMarkWorkerIdle()`，这三种模式分别对应了**全职模式**、**根据比例做标记，留一部分 CPU 给其他协程**以及只在 **CPU 空闲的时候进行标记**。
```Go
func gcBgMarkWorker(ready chan struct{}) {
	// ...
	for {
		// ...
		gopark(func(g *g, nodep unsafe.Pointer) bool {
			// ...
			casGToWaitingForGC(gp, _Grunning, waitReasonGCWorkerActive)
			switch pp.gcMarkWorkerMode {
			default:
				throw("gcBgMarkWorker: unexpected gcMarkWorkerMode")
			case gcMarkWorkerDedicatedMode:
				// ...
				// 全职
				gcDrainMarkWorkerDedicated(&pp.gcw, false)
			case gcMarkWorkerFractionalMode:
				// 分时复用
				gcDrainMarkWorkerFractional(&pp.gcw)
			case gcMarkWorkerIdleMode:
				// 空闲时标记
				gcDrainMarkWorkerIdle(&pp.gcw)
			}
			casgstatus(gp, _Gwaiting, _Grunning)
		})
}
```
三者底层都调用了 `gcDrain()` 方法。
```Go
//go:nowritebarrier
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	if !writeBarrier.enabled {
		throw("gcDrain phase incorrect")
	}

	// ...

	// Root 对象标记
	if work.markrootNext < work.markrootJobs {
		// Stop if we're preemptible, if someone wants to STW, or if
		// someone is calling forEachP.
		for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			markroot(gcw, job, flushBgCredit)
			if check != nil && check() {
				goto done
			}
		}
	}
	
	// 循环扫描堆对象 直到退出或被抢占
	for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
		// ...
		scanobject(b, gcw)
		// ...
	}

done:
	// Flush remaining scan work credit.
	if gcw.heapScanWork > 0 {
		gcController.heapScanWork.Add(gcw.heapScanWork)
		if flushBgCredit {
			gcFlushBgCredit(gcw.heapScanWork - initScanWork)
		}
		gcw.heapScanWork = 0
	}
}
```
实际上底层就是  `markroot()` 和 `scanstack()` 两个方法，分别标记根对象和堆对象，这里就不展开二者的源码了。
`markroot():
1. 扫描已经初始化的全局变量 .data 段
2. 扫描未初始化的全局变量 .bss 段
3. 扫描各个协程的栈
其实上述三个部分就是主要对应了所谓的 Root 对象
在标记的时候，都会走向 `greyobject()` 方法，进行真正的将对象标记为灰色。
```Go
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {
	// obj should be start of allocation, and so must be at least pointer-aligned.
	if obj&(goarch.PtrSize-1) != 0 {
		throw("greyobject: obj not pointer-aligned")
	}
    // 获取 obj 对应的 mspan 对应的位图，用于查看 obj 是不是已经被标记了
	mbits := span.markBitsForIndex(objIndex)
    
    // 如果已经被标记过，则跳过
	if useCheckmark {
		if setCheckmark(obj, base, off, mbits) {
			// Already marked.
			return
		}
	} else {
        // 如果当前 GC 尝试标记一个未分配内存或者已经释放的对象，则报错
		if debug.gccheckmark > 0 && span.isFree(objIndex) {
			print("runtime: marking free object ", hex(obj), " found at *(", hex(base), "+", hex(off), ")\n")
			gcDumpObject("base", base, off)
			gcDumpObject("obj", obj, ^uintptr(0))
			getg().m.traceback = 2
			throw("marking free object")
		}

        // 如果还没被标记，则标记
		if mbits.isMarked() {
			return
		}
		mbits.setMarked()
        
        // 在对应的页上标记
		// Mark span.
		arena, pageIdx, pageMask := pageIndexOf(span.base())
		if arena.pageMarks[pageIdx]&pageMask == 0 {
			atomic.Or8(&arena.pageMarks[pageIdx], pageMask)
		}
	}
    
    // 如果是一个 noscan 对象，则直接染黑后返回
	if span.spanclass.noscan() {
		gcw.bytesMarked += uint64(span.elemsize)
		return
	}
	// ...
	sys.Prefetch(obj)
	// Queue the obj for scanning.
	if !gcw.putFast(obj) {
		gcw.put(obj)
	}
}
```
循环标记完后，会进入 `gcMarkDone()`，一系列校成功束后，会结束整个标记阶段。至此，GC 的标记阶段也就结束了
## GC - Sweep
	runtime/mgcsweep.go
gcMarkDone() 后会调用 gcMarkTermination() 后会调用 gcSweep() 从而进入真正的清扫逻辑。
这里闲了再做展开，流程大致是：
1. 遍历 mspan
2. 检查标记位（要回收的对象会被标记为 0，代表不存活）
3. 将回收对象加入 mspan 的 free list
# Java GC 简述
首先 JVM 的 GC 是基于分代理论的，Java 设计团队认为，大多数对象都是朝生暮死，很多对象的生命周期都很短，所以将对象分为了 新生代、老年代以及永久代，固存储区域也就被氛围了新生代区、老年代区以及永久代区，其中新生代区又进一步划分为 Edan区、S0 和 S1区，其中的对象生命周期依次增长。<br>
Java 常用的垃圾回收器有两种主要算法：标记-复制以及标记-整理，这里不赘述其中的区别。
## Java常见的垃圾回收器整理
1. 串行垃圾回收器
	1. Serial & Serial Old
	2. Serial 采用标记-复制、Serial Old 采用标记-整理
2. 并发垃圾回收器
	1. Parallel & Parallel Old & ParNew & CMS
	2. Parallel 采用标记-复制、Parallel-Old 采用标记-整理（利用并行提高 GC 效率和降低用户感知）
	3. ParNew 新生代采用标记-复制，其主要用于新生代，搭配CMS 老年代回收器使用
	4. CMS 老年代采用标记-整理（降低了 STW 的时间）
3. G1 垃圾回收器
	1. 将堆内存当做一个整体分区，然后分成独立的 Block，每个 Block 都可以视为任何一类分区，既可以是 Eden 也可以是 Old
	2. 首先进行新生代回收（STW），然后并发标记（重新标记 STW），最后混合回收
	3. 新生代回收主要是将新生代收集整理到一个新的 Young 分区内，将年长的对象整理到一个 Old 区内，然后回收掉垃圾
	4. 当堆内存使用量超过 45%，触发并发标记，标记后会根据暂停时间目标收集存活对象少的Old 区
	5. 混合回收阶段会和新生代回收类似，将 Edan 区对象整理到到一个新 S 区，将 Old 对象整理到一个新的 Old 区
	6. 当回收对象的速度 ＜ 分配新对象的速度，就会导致垃圾回收失败，进行一次 Full-GC

# Why Go 没有采用分代理念？
首先 Go 的设计理念就是大道至简，如果要分代，则必然会带来额外的复杂开销，所以是否值得呢？<br>
1. Go 中存在内存逃逸分析，会将生命周期比较长的对象划分到堆上去，等于天然进行了分代，所以再去分代意义不大
2. Go 的定位是云原生、服务端等业务场景，更倾向于使用短生命周期对象（比如说网络请求上下文），这些对象很多时候都是在栈上分配的，不会进入堆
3. Go 团队相较于吞吐量，可能更看重 STW 时间
下面贴一些关于 Go-GC 设计理念的讨论
4. https://go.dev/blog/ismmkeynote
5. https://itnext.io/go-does-not-need-a-java-style-gc-ac99b8d26c60
6. https://blog.plan99.net/modern-garbage-collection-911ef4f8bd8e
7. https://groups.google.com/g/golang-nuts/c/KJiyv2mV2pU?utm_source=chatgpt.com&pli=1

# 总结
Go 的 Gc 的追求的是极致短的 STW，外加并发标记的特点，尽可能让用户感知不到 Gc 的影响<br>
最近我一直在思考，Go 和 Java 的 Gc 各适用什么场景？其实有些钻牛角尖了，Java 得益于其框架的完备以及 JVM 的各种调优手段，使得其拥有很高的上限，但也带来了更大的学习带孩，所以 Java 可能更加适合大型的企业级开发，或者说更加多变的场景；而 Go 其本身有着相对来说不错的 Gc 性能外加其协程的方便使用和性能，使其可以开箱即用，不需要考虑调优，所以可能更加适合网关这种多网络连接的场景。