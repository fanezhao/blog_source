---
title: Go并发编程（十一）调度器
date: 2020-05-18 20:00:00
tags:
  - Go
---

> 上一章讲过，两级线程模型中的一部分调度任务会由操作系统内核之外的程序承担。在Go中，调度器因负责这一部分调度任务。

调度器的主要对象是M、P和G的实例，调度器的辅助设施包括上篇介绍过的各种核心元素的容器。其实，每个M（即每个内核线程）在运行过程中都会按需执行一些调度任务。不过，为了更加容易理解，我把这些调度任务统称为“调度器的调度行为”。

### 基本结构

调度器有自己的数据结构，其中就有我们已知熟知的空闲M列表、空闲P列表、可运行G队列和自由G列表。下面还有几个重要字段。

| 字段名称   | 数据类型 | 用途简述                                 |
| ---------- | -------- | ---------------------------------------- |
| gcwaiting  | uint32   | 表示是否需要因一些任务而停止调度         |
| stopwait   | int32    | 表示需要停止但仍未停止的P的数量          |
| stopnote   | note     | 用于实现与stopwait相关的事件通知机制     |
| sysmonwait | uint32   | 表示在停止调度期间系统监控任务是否在等待 |
| sysmonnote | note     | 用于实现与sysmonwait相关的事件通知机制   |

在这张表中的字段都与需要停止调度的任务有关。在Go运行时系统中，一些任务在执行前是需要暂停调度的，例如垃圾回收任务中的某些子任务，又比如发起运行时恐慌（panic）的任务。我们暂且把这类任务称为**串行运行时任务**。

字段`gcwaiting`、`stopwait`和`stopnote`都是串行运行时任务执行前后的辅助协调手段。

- `gcwaiting`字段的值用于表示是否需要停止调度：在停止调度前，该值会被置为1；在恢复调度前，该值会被置为0。这样做的目的是，一些调度任务在执行时只要发现`gcwaiting`的值为1，就会把该P的状态置为`Pgcstop`，然后自减`stopwait`的值。如果自减后的值为0，就说明所有P的状态都已为`Pgcstop`。这时就可以利用`stopnote`字段，唤醒因等待调度停止而暂停的串行运行时任务了。

- `sysmonwait`和`sysmonnote`与前面那组用途类似，区别是它们针对的是系统监测任务。**在串行运行时任务执行之前，系统监测任务也需要暂停**。`sysmonwait`为1时表示已暂停，为0表示未暂停。系统监测任务是持续执行的，它处在无尽的循环之中。在每次迭代之初，系统监测任务都会先检查调度情况。一旦发现调度停止（`gcwaiting`不为1或所有P都已闲置），就会把`sysmonwait`置为1，并利用`sysmonnote`字段暂停自身；另外，在恢复调度之前，调度器若发现`sysmonwait`值不为0，就会把它置为0，并利用`sysmonnote`字段恢复系统监测任务的执行。

上述5个调度器字段都是为了串行运行时任务而存在的。并且，运行时系统一定会保证操作它们时的并发安全。

## 一轮调度

上一章我们提到过，引导程序会为Go程序运行建立必要的环境。在引导程序完成一系列初始化工作之后，Go程序的`main`函数才会真正的执行。

引导程序会在最后让调度器进行**一轮调度**，这样才能够让封装了` main`函数的G马上有机会运行。封装`main`函数的G总是Go运行时系统创建的第一个**用户G**（用于封装用户级别的程序片段，即需并发执行的函数）。相对的，用于封装运行时任务的G称为**运行时G**。

下面我们深入了解一下调度器在一轮调度中都做了哪些工作。

{% qnimg 一轮调度.jpg %}

一轮调度，由Go标准库代码包`runtime`中的`schedule`函数代表。

在一轮调度开始处，调度器会先判断当前M是否已被锁定。M和G是可以成对的锁定在一起的。锁定M和G的操作可以说是为CGO准备的。

> CGO代表了Go中的一种机制，是Go程序和C程序中的一座桥梁，使它们的互相调用成为可能。

接着，再判断当前M是否已与某个G锁定。如果有，就会立即停止调度并停止当前M（或暂时阻塞）。一旦与它锁定的G处于可运行状态，它就会被唤醒并继续运行那个G。

如果当前M未与任何G锁定。这时，调度器会检查是否有串行运行时任务正在等待执行。这类任务在执行时需要停止调度器。官方称之为“Stop The World”，简称`STW`。还记得`gcwaiting`字段么？如果`gcwaiting`字段值不为0，那一轮调度就会进入另一个分支：停止并阻塞当前M以等待运行时串行任务执行完成。一旦串行任务执行完成，该M就会被唤醒，一轮调度也会再次开始。

最后，如果调度器在此关于锁定和串行运行时任务的判断都为假，就会开始真正的可运行G之旅。一旦找到一个G，调度器就会在判定该G未与任何M锁定之后，立即让当前M运行它。

调度器会先从一些比较容易找到可运行G的地方入手，即：全局的（或称调度器的）可运行G队列和本地P的可运行G队列。如果这些地方找不到，调度器会进入强力查找模式，即图中的子流程：**“全力查找可运行的G”**。

如果经过一番强力查找还是不能找到可运行的G，该子流程就会暂停，直到在可运行的G出现才会继续下去。也就是说，这个全力查找可运行的G的子流程结束，就意味着当前M抢到了一个可运行的G。

### 一轮调度在什么情况会被触发？

一轮调度是Go调度器最核心的流程，在很多情况下都会被触发。例如：

- 在用户程序启动时的一系列初始化工作之后，一轮调度流程会首次启动并使封装`main`函数的那个G被调度运行。
- 又例如，某个G运行的阻塞、结束、退出系统调用，以及栈的增长都会使调度器进行一轮调度。
- 还有，用户程序对某些标准库函数的调用也会触发一轮调度。比如，对`runtime.Gosched`函数的调用会使当前G的暂停运行，并让出CPU给其它的G。又比如，调用`runtime.Goexit`函数会结束当前G的运行，同时也会进行一轮调度。

### 全力查找可运行的G

上面说到，如果调度器没有找到可运行的G，就会进入“全力查找可运行的G”的子流程。概括的讲，这个子流程会多次尝试从各处搜索可运行的G，甚至还会从别的P那时偷取可运行的G。它由`runtime.findrunnable`函数表示，该函数会返回一个`Grunnable`状态的G。

其中的搜索流程大致分为2个阶段和10个步骤，具体如下，这里直接通过源码体现：

```go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// 这里的条件和handoffp中的条件必须一致：如果findrunnable将返回G以运行，handoffp必须以M开始。

top:
	_p_ := _g_.m.p.ptr()
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if _p_.runSafePointFn != 0 {
		runSafePointFn()
	}

	now, pollUntil, _ := checkTimers(_p_, 0)
	// 1、获取执行终结器的G
	if fingwait && fingwake {
		if gp := wakefing(); gp != nil {
			ready(gp, 0, true)
		}
	}
	if *cgo_yield != nil {
		asmcgocall(*cgo_yield, nil)
	}

	// 2、从本地P的可运行G队列中获取G
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// 3、从调度器的可运行G队列中获取G
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// 4、从网络io轮询器（或称netpoller）处获取G
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// 5、从其它P的可运行G队列获取G
	procs := uint32(gomaxprocs)
	ranTimer := false
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
	for i := 0; i < 4; i++ {
		// ......
	}
	if ranTimer {
		// Running a timer may have made some goroutine ready.
		goto top
	}

stop:

	// 6、获取执行GC标记任务的G
	if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
		gp := _p_.gcBgMarkWorker.ptr()
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}

	// ......
    
	// 7、从调度器的可运行G队列中获取G
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	pidleput(_p_)
	unlock(&sched.lock)

	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}
	}

	// 8、从全局P列表中的每个P的可运行G队列获取G
	for _, _p_ := range allpSnapshot {
		if !runqempty(_p_) {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				if wasSpinning {
					_g_.m.spinning = true
					atomic.Xadd(&sched.nmspinning, 1)
				}
				goto top
			}
			break
		}
	}

	// 9、获取执行GC标记任务的G
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(nil) {
		lock(&sched.lock)
		_p_ = pidleget()
		if _p_ != nil && _p_.gcBgMarkWorker == 0 {
			pidleput(_p_)
			_p_ = nil
		}
		unlock(&sched.lock)
		if _p_ != nil {
			acquirep(_p_)
			if wasSpinning {
				_g_.m.spinning = true
				atomic.Xadd(&sched.nmspinning, 1)
			}
			// Go back to idle GC check.
			goto stop
		}
	}

	// 10、从网络io轮询器（或称netpoller）处获取G
	if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		// ......
	}
	stopm()
	goto top
}
```

如果经过上面这10个步骤依然没有找到可运行的G，调度器就会停止当前的M。在之后的某个时刻，该M被唤醒后，它会重新进入“全力查找可运行的G”的子流程。

对这个子流程还有几个细节说明一下：

- 网络I/O（即`netpoller`）是Go为了在操作系统提供的异步I/O基础组件之上，实现自己的阻塞式I/O而编写的一个子程序。
- 在第5步，调度器从其它P的可运行队列获取G是有条件的。
  - 1、除了本地P还有非空闲的P，如果都是空闲的P，就没必要再去它们那查找可运行的G了。
  - 2、当前M正处于自旋状态，或者处于自旋状态的M数量小于非空闲P的数量的1/2。这主要是为了控制自旋M的数量，过多的自旋M会消耗太多CPU资源。没有必要让更多的M去查找可运行的G。

说到**自旋状态**，它实际是标识了M的一种工作状态。M处于自旋状态，意味着它还没找到G来运行，无论是由于找到了可运行的G，还是由于因始终未找到可运行的G而需要停止M，当前M都会退出自旋状态。

一般情况下，运行时系统中至少会有一个自旋状态的M，调度器会尽量保证有一个自旋M的存在。除非发现没有自旋的M，调度器是不会新启用或恢复一个M去运行新的G。一旦需要新启用或恢复一个M，它最初总是会处于自旋状态。

## 启用或停止M

上面多次提到，调度器有时会停止当前M。在Go标准库代码包`runtime`中，有下面这样几个函数负责M的启用或停止。

- **stopm()**。停止当前M的运行，直到因有新的G变得可运行而被唤醒。
- **gcstopm()。**为串行运行时任务的执行让路，停止当前M的执行。串行运行时任务执行完毕后会被唤醒。
- **stoplockedm()**。停止已与某个G锁定的当前M的执行，直到因这个G变得可运行而被唤醒。
- **startlockedm(gp *g)**。唤醒与gp锁定的那个M，并让该M去执行gp。
- **startm(*_*p*_* *p, spinning bool)**。唤醒或创建一个M去关联*_*p*_*并开始执行。

下图是Go调度器对这些函数的使用，

{% qnimg 启用和停止M.jpg %}

1、调度器在调度流程的时候，会先检查当前M是否与某个G锁定。如果锁定，调度器就会调用`stoplockedm`函数停止当前M。`stoplockedm`函数会先解除当前M与本地P之间的关联，并通过调用一个叫`handoffp`的函数把这个P转手给其它M，在这个转手的过程中会间接调用`startm`函数。一旦这个P被转手，`stoplockedm`就会停止当前M的执行，并等待唤醒。

2、如果调度程序为当前M找到了一个可运行的G，却发现该G已与某个M锁定了，那么就会调用`startlockedm`函数并把这个G作为参数传入。`startlockedm`函数会通过`gp`的`lockedm`字段找到与之前锁定的那个M（以下简称“已锁M”），并把当前M的本地P转手给它。这里的转手P的过程要比过程1中简单的多，`startlockedm`函数会先解除当前M与本地P的关联，然后把这个P赋给已锁M的`nextp`字段（即预联它们）。

3、`startlockedm`函数会的执行会使其参数gp绑定的那个M（已锁M）被唤醒。通过`gp`的`lockedm`字段可以找到已锁M。一旦已锁M被唤醒，就会与和它预联的P产生正式关联，并去执行与之相关的G。

4、`startlockedm`函数在最后会调用`stopm`函数。`stopm`函数会先把当前M放入调度器的空闲M列表，然后停止当前M。这里被停止的M，可能会在之后因有P需要转手，或有G需要执行而被唤醒。

纵观上述流程，我们可以看到因M和G的锁定而执行的分支流程。**这里涉及两个M，一个是因等待执行与之锁定的G而停止的M，一个是获取到一个已锁定的G却不能执行的M**。

从另一个角度看，一旦M要停止就会把它的本地的P转手给别的M。一旦M被唤醒，就会先找到一个P与之关联，即找到它的新的本地P。并且，这个P一定是在该M被唤醒之前由别的M预联给它的。因此P问题会被高效利用。如果`handoffp`函数无法把作为其参数的P转给一个M，那么就会把这个P放入调度器的空闲P列表。

5、调度器在执行调度流程的时候，也会检查是否有串行运行时任务正在等待执行。如果有，调度器`gcstopm`函数停止当前M。`gcstopm`函数会先通过当前M的`spinning`字段检查它的自旋状态，如果状态为`true`，就把`false`赋给它，然后把调度器中用于自旋M数量的`nmspinning`字段减1。如此一来就完全重置了当前M的自旋状态标识，一个将要停止的M理应脱离自旋状态。在这之后，`gcstop`函数会释放本地P，并将其状态置为`Pgcstop`。然后再去自减并检查调度器的`stopwait`字段，并在发现`stopwait`字段的值为0时，通过调度器的`stopnote`字段唤醒等待执行的串行运行时任务。

6、`gcstopm`函数会在最后调用`stopm`函数。同样的，当前M会被放入调度器的空闲M列表并停止。只要串行运行时任务准备执行，“Stop the world”就会开始，所有在调度过程中的M就都会执行步骤5和6。其中的步骤5更是决定了串行运行时任务是否能够被尽早地执行。

7、调度总有不成功的时候。经过完整的一轮调度之后，仍找不到一个可运行的G给当前M执行，那么调度程序就会通过调用`stopm`函数停止当前M。换句话说，这时已经没有多余的工作可以做了，为了节省资源就要停掉一些M。

8、所有经由调用`stopm`函数停止的M，都可以通过调用`startm`函数唤醒，与步骤7对应，一个M被唤醒的原因总是有新的工作要做。比如，有了新的自由的P，或者有了新的可运行的G。

有时候，传入`startm`函数的参数*_*p*_*为`nil`，这就说明在唤醒一个M的同时，需要从调度器的空闲P列表获取一个P作为M运行G的上下文环境。如果这个列表也为空，这里`startm`会直接返回。

一旦有了一个新的P，`startm`函数就会再次从调度器的空闲M列表获取一个M；如果该列表已空就创建一个新的M。无论如何，`startm`函数都会把拿到的P和这个M预联，然后让该M做好执行准备。

在高并发的Go程序中，启停M的流程在调度器中经常被执行到。因为并发量越大，调度器对M、P和G的调度就越频繁。各个G总是通过这样或那样的途径用到系统调用，也经常会使用到Go本身的各种组件（如channel、Timer等）。这些都直接或间接的涉及M的启停。理解此流程能够更好的帮助我们了解Go调度器的运行机制。

## 总结

本篇文章，我们主要学习了Go中最核心的内容之一——调度器的调度原理。在程序启动的时候，引导程序会为Go程序的运行建立必要的环境，并在最后进行调度器的一轮调度。除此之外，我们还讲述了一轮调度的基本流程和触发场景。在最后还学习M的启停流程以加深对一轮调度的理解。下面是一轮调度的源码：

```go
// 一轮调度
func schedule() {
	_g_ := getg()

    // 判断当前M是否被锁定
	if _g_.m.locks != 0 {
		throw("schedule: holding locks")
	}

    // 判断当前M是否与某个G锁定。
	if _g_.m.lockedg != 0 {
        // 如果锁定，就会立即停止当前M（或者说暂时阻塞）。
        // stoplockedm函数会先解除当前M与本地P的关联，并通过一个叫handoffp的函数把这个P转手给其它M。
        // 一旦这个P被转手，stoplockedm函数就会停止当前M的运行，并等待唤醒。
		stoplockedm()
		execute(_g_.m.lockedg.ptr(), false) // Never returns.
	}

	// We should not schedule away from a g that is executing a cgo call,
	// since the cgo call is using the m's g0 stack.
	if _g_.m.incgo {
		throw("schedule: in cgo")
	}

top:
	pp := _g_.m.p.ptr()
	pp.preempt = false

    // 检查是否有串行运行时任务正在等待执行。
	if sched.gcwaiting != 0 {
        // 停止并阻塞当前M以等待运行时串行任务执行完成。
        // 一旦串行任务执行完成，该M就会被唤醒，一轮调度也会再次开始。
		gcstopm()
		goto top
	}
	if pp.runSafePointFn != 0 {
		runSafePointFn()
	}

	// Sanity check: if we are spinning, the run queue should be empty.
	// Check this before calling checkTimers, as that might call
	// goready to put a ready goroutine on the local run queue.
	if _g_.m.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}

	checkTimers(pp, 0)

	var gp *g
	var inheritTime bool

    tryWakeP := false
    // 试图获取执行踪迹读取任务的G
	if trace.enabled || trace.shutdown {
		gp = traceReader()
		if gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			traceGoUnpark(gp, 0)
			tryWakeP = true
		}
	}
	if gp == nil && gcBlackenEnabled != 0 {
        // 试图获取执行GC标记任务的G
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		tryWakeP = tryWakeP || gp != nil
	}
	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            // 在特定条件下从全局可运行G队列获取G
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
        // 从本地P的可运行队列获取G
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
        // 进入强力查找模式，全力查找可运行的G。
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// This thread is going to run a goroutine and is not spinning anymore,
	// so if it was marked as spinning we need to reset it now and potentially
	// start a new spinning M.
	if _g_.m.spinning {
		resetspinning()
	}

	if sched.disable.user && !schedEnabled(gp) {
		// Scheduling of this goroutine is disabled. Put it on
		// the list of pending runnable goroutines for when we
		// re-enable user scheduling and look again.
		lock(&sched.lock)
		if schedEnabled(gp) {
			// Something re-enabled scheduling while we
			// were acquiring the lock.
			unlock(&sched.lock)
		} else {
			sched.disable.runnable.pushBack(gp)
			sched.disable.n++
			unlock(&sched.lock)
			goto top
		}
	}

	// If about to schedule a not-normal goroutine (a GCworker or tracereader),
	// wake a P if there is one.
	if tryWakeP {
		if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
			wakep()
		}
    }
    
    // 当调度器为当前M找到一个可运行的G，但却发现该G己与某个M锁定。
	if gp.lockedm != 0 {
        // 就唤醒那个与之锁定的M以运行该G，并重新为当前M寻找一个可运行的G。
        // startlockedm函数会通过gp的lockedm字段找到与之锁定的那个M（已锁M），然后解除当前M和本地P之间的关联，
        // 然后把这个P赋给已锁M的nextp字段（预联它们），完成本地P的转手。一旦已锁M被唤醒，就会与和它预联的P产生关联，并去执行与之关联的G。
        // startlockedm最后还会去调用stopm函数。stopm函数会选择当前M放入调度器的空闲M列表，然后停止当前M。
        // 这里被停止的M，可能会在之后有因有P需要转手，或有G需要执行而被唤醒
		startlockedm(gp)
		goto top
	}

    // 让当前M运行G
	execute(gp, inheritTime)
}
```

