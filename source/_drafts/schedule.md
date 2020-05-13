---
title: schedule
tags:
---

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

    // 检查是否有暗串行任务正在等待执行。
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
    // 试图获取反选踪迹读取任务的G
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