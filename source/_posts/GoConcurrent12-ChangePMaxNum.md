---
title: Go并发编程（十二）修改P的最大数量
date: 2020-05-22 18:13:02
tags:
  - Go
---

> 在[Go线程实现模型](http://zmoyi.com/2020/05/09/GoConcurrent10-GoThreadModel/)中，P起到了承上启下的作用，P的最大数量的变更就意味着要改变G运行的上下文环境，这种变更也直接影响着G程序的并发性能。本章来说一下其中的流程。

## 修改P的最大数量

**在默认情况下，P的最大数量等于正在运行当前Go程序的机器的CPU核心的数量**。一台计算机CPU核心的数量，说明了它能够在同一时刻执行多少个程序指令。我们可以通过调用`runtime.GOMAXPROCS`函数改变这个最大数量，但是这样做有时会有较大损耗。

当我们调用`runtime.GOMAXPROCS`函数时，它会先进行下面两项检查，以确保变更合法和有效。

- 如果传入的参数值（新值）比运行时系统为此设定的硬性上限值（256）大，且无论传入的新值有多大，最终的值也不会超过256。这是运行时系统对自身的一种保护。
- 如果新值不是正整数，或者与存储在运行时系统中的P最大数量值（旧值）相同，那么该函数会直接忽略变更而直接返回旧值。

一旦通过这两项检查，该函数会先通过调度器停止一切调度工作（Stop The World），然后它会暂存新值、重启调度工作（Start The World），最后将旧值作为结果返回，重启调度工作由`runtime.startTheWorldWithSema`函数完成。

```go
func startTheWorldWithSema(emitTraceEvent bool) int64 {
    mp := acquirem() // disable preemption because it can be holding p in a local var
    if netpollinited() {
        list := netpoll(0) // non-blocking
        injectglist(&list)
    }
    lock(&sched.lock)

    procs := gomaxprocs
    if newprocs != 0 {
        procs = newprocs
        newprocs = 0
    }
    // P的最大数量变更流程
    p1 := procresize(procs)
    sched.gcwaiting = 0
    if sched.sysmonwait != 0 {
        sched.sysmonwait = 0
        notewakeup(&sched.sysmonnote)
    }
    unlock(&sched.lock)
	// 遍历拥有可运行G的P的列表，保证里面的一定能与一个M产生关联。
    for p1 != nil {
        p := p1
        p1 = p1.link.ptr()
        if p.m != 0 {
            mp := p.m.ptr()
            p.m = 0
            if mp.nextp != 0 {
                throw("startTheWorld: inconsistent mp->nextp")
            }
            mp.nextp.set(p)
            notewakeup(&mp.park)
        } else {
            // Start M to run P.  Do not start another M below.
            newm(nil, p)
        }
    }

    // Capture start-the-world time before doing clean-up tasks.
    startTime := nanotime()
    if emitTraceEvent {
        traceGCSTWDone()
    }
    
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
        wakep()
    }

    releasem(mp)

    return startTime
}

```



在调度工作真正被重启之前，调度器如果发现有新值暂存，那么就会进入P的最大数量的变更流程。此流程由`runtime`包的`procresize`函数实现。

## P的最大数量的变更流程

```go
func procresize(nprocs int32) *p {
	old := gomaxprocs
    // 新旧值合法性检查
	if old < 0 || nprocs <= 0 {
		throw("procresize: invalid arg")
	}

	// ......

	// 全局P列表扩容
	if nprocs > int32(len(allp)) {
		lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// 新建P填充全局P列表
	for i := old; i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {
			pp = new(p)
		}
		pp.init(i)
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}

	// 当前M不能没有P，所以程序会试图把该M之前的P还给它，若发现那个P已经被清理，就把全局列表中的第一个P给它。
	_g_ := getg()
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
		_g_.m.p.ptr().status = _Prunning
		_g_.m.p.ptr().mcache.prepareForSweep()
	} else {
		if _g_.m.p != 0 {
			if trace.enabled {
				traceGoSched()
				traceProcStop(_g_.m.p.ptr())
			}
			_g_.m.p.ptr().m = 0
		}
		_g_.m.p = 0
		_g_.m.mcache = nil
		p := allp[0]
		p.m = 0
		p.status = _Pidle
		acquirep(p)
		if trace.enabled {
			traceGoStart()
		}
	}

	// 清理无用的P，将其设置成Pdead状态
	for i := nprocs; i < old; i++ {
		p := allp[i]
		p.destroy()
	}

	// Trim allp.
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		unlock(&allpLock)
	}
	// 最后，程序会再检查一遍前N个P。如果它的可运行G队列为空，就把它放入调度器的空闲P列表，否则就试图拿一个M与之绑定，然后把它放入本地的可运行P列表。
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
		if runqempty(p) {
			pidleput(p)
		} else {
			p.m.set(mget())
			p.link.set(runnablePs)
			runnablePs = p
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
	return runnablePs
}

```



在此流程中，旧值会先被获取。如果发现旧值或新值不合法，程序就会发起一个运行时`panic`，程序终止。不过由于`runtime.GOMAXPROCS`函数已做过检查，所以这里永远不会发生。

在通过对旧值和新值检查之后，程序会对全局P列表中的前*i*个P进行检查和必要的初始化。**这里的*i*代表新值**。如果全局P列表中的P的数量不够，程序还会新建相应数量的P，并把它们追加到全局P列表中，新P的状态为`Pgcstop`，以表示它还不能使用。

> BTW，全局P列表中的所有P的可运行G队列的固定长度都是256。如果这个队列满了，程序就会把其中半数的可运行G转移到调度器的可运行G队列中。

在完成对前*i*个P的重新设置之后，程序对全局P列表中的第*i*+1个到*j*个P（如果有的话）进行清理。**这里*j*代表旧值**。

其中，最重要的工作就是把这些P的可运行G队列中的G及其`runnext`字段中的G（如果有的话）全部取出，并依次放入调度器的可运行G队列中。程序也会 试图获取这些P持有的GC标记专用G，若取到，同样放入调度器的可运行G队列。此外，程序还会把这些P的自由G列表中的G，转移到调度器的自由G列表中。

然后，这些P都会被设置成`Pdead`状态，以便之后进行销毁。之所以不能直接销毁它们，是因为它们可能会这被正在进行系统调用的M引用。如果某个P被这样的M引用但却被销毁了，就会在该M完成系统调用的时候造成错误。

至此，全局P列表中的P都已被重置，这也包括了与执行`procresize`函数当前M关联的那个P。当前M不能没有P，所以程序会试图把该M之前的P还给它，若发现那个P已经被清理，就把全局列表中的第一个P给它。

最后，程序会再检查一遍前*N*个P。如果它的可运行G队列为空，就把它放入调度器的空闲P列表，否则就试图拿一个M与之绑定，然后把它放入本地的可运行P列表。这样就筛选出一个拥有可运行G的P的列表，`procresiaze`函数会把这个列表作为结果值返回。负责重启调度工作的程序会先检查这个列表中的P，以保证它们一定能与一个M产生关联。随后，程序会让与这些P关联的M都运作起来。

以上就是变更P的最大数量发生的事情，下面是其核心流程图。

{% qnimg 变更P最大数量的核心流程.jpg %}

再次强调，虽然可以通过`runtime.GOMAXPROCS`函数改变运行时系统中P的最大数量，但是它们会引起调度工作的暂停。对于响应时间敏感的Go程序，这种暂停会带来一定的性能影响。所以请牢记此函数的正确使用方式（参考[Go并发编程模型](http://zmoyi.com/2020/05/09/GoConcurrent10-GoThreadModel/))）。

## 总结

综上，P的最大数量的变更对G的上下文环境影响很大，修改P的方式有两种：

- 调用函数`runtime.GOMAXPROCS`传入想要设定的数量。
- 在Go程序运行前设置环境变量`GOMAXPROCS`的值。

后一种对程序影响比较小。这里我们主要学习第一种修改方式的变更流程和可能会对程序造成的影响。并在最后强调了通过这种方式修改P的最大数量的正确方式。

