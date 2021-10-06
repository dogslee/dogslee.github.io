---
title: golang运行时核心调度函数(schedule，findrunnable，sysmon)源码分析
date: 2021-10-06 11:35:02
index_img: /img/golang/go.jpg
banner_img:
categories: 
- golang
tags:
- runtime
- schedule
- findrunnable
- sysmon
---

# 概述

分析核心调度函数 shcedule findrunnable sysmon

## schedule



dlv调试：

```shell
(dlv) b runtime.schedule
Breakpoint 1 set at 0x435cf3 for runtime.schedule() /usr/lib/golang/src/runtime/proc.go:2609
(dlv) c
> runtime.schedule() /usr/lib/golang/src/runtime/proc.go:2609 (hits total:1) (PC: 0x435cf3)
Warning: debugging optimized function
  2604:         }
  2605: }
  2606:
  2607: // One round of scheduler: find a runnable goroutine and execute it.
  2608: // Never returns.
=>2609: func schedule() {
  2610:         _g_ := getg()
  2611:
  2612:         if _g_.m.locks != 0 {
  2613:                 throw("schedule: holding locks")
  2614:         }
```

全部的源码内容如下：

```go
func schedule() {
	_g_ := getg()

	if _g_.m.locks != 0 {
		throw("schedule: holding locks")
	}

	if _g_.m.lockedg != 0 {
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

	if sched.gcwaiting != 0 {
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

	// Normal goroutines will check for need to wakeP in ready,
	// but GCworkers and tracereaders will not, so the check must
	// be done here instead.
	tryWakeP := false
	if trace.enabled || trace.shutdown {
		gp = traceReader()
		if gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			traceGoUnpark(gp, 0)
			tryWakeP = true
		}
	}
	if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		tryWakeP = tryWakeP || gp != nil
	}
	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		// We can see gp != nil here even if the M is spinning,
		// if checkTimers added a local goroutine via goready.
	}
	if gp == nil {
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
		wakep()
	}
	if gp.lockedm != 0 {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp)
		goto top
	}

	execute(gp, inheritTime)
}
```

忽略GC和trace后分析源码发现shedule函数本质就是尽力找到可运行的g，然后去运行g上面的任务函数。查找g的流程如下：

1. 如果当前GC需要停止整个世界（STW), 则调用gcstopm休眠当前的M
2. 每隔61次调度轮回从全局队列找，避免全局队列中的g被饿死。
3. 从p.runnext获取g，从p的本地队列中获取。
4. 调用 `findrunnable` 找g，找不到的话就将m休眠，等待唤醒。

当找到一个g后，就会调用 `execute` 去执行g。



## findrunnable

dlv调试：

```shell
[root@583d9a8ec1db p1]# dlv exec ./main
Type 'help' for list of commands.
(dlv) b runtime.findrunnable
Breakpoint 1 set at 0x4348d8 for runtime.findrunnable() /usr/lib/golang/src/runtime/proc.go:2189
(dlv) c
> runtime.findrunnable() /usr/lib/golang/src/runtime/proc.go:2189 (hits total:1) (PC: 0x4348d8)
Warning: debugging optimized function
  2184:         gogo(&gp.sched)
  2185: }
  2186:
  2187: // Finds a runnable goroutine to execute.
  2188: // Tries to steal from other P's, get g from local or global queue, poll network.
=>2189: func findrunnable() (gp *g, inheritTime bool) {
  2190:         _g_ := getg()
  2191:
  2192:         // The conditions here and in handoffp must agree: if
  2193:         // findrunnable would return a G to run, handoffp must start
  2194:         // an M.
(dlv) si
> runtime.findrunnable() /usr/lib/golang/src/runtime/proc.go:2189 (PC: 0x4348df)
Warning: debugging optimized function
  2184:         gogo(&gp.sched)
  2185: }
  2186:
  2187: // Finds a runnable goroutine to execute.
  2188: // Tries to steal from other P's, get g from local or global queue, poll network.
=>2189: func findrunnable() (gp *g, inheritTime bool) {
  2190:         _g_ := getg()
  2191:
  2192:         // The conditions here and in handoffp must agree: if
  2193:         // findrunnable would return a G to run, handoffp must start
  2194:         // an M.
```



全部源代码如下：

```go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

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

	if fingwait && fingwake {
		if gp := wakefing(); gp != nil {
			ready(gp, 0, true)
		}
	}
	if *cgo_yield != nil {
		asmcgocall(*cgo_yield, nil)
	}

	// local runq
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
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

	// Steal work from other P's.
	procs := uint32(gomaxprocs)
	ranTimer := false
	// If number of spinning M's >= number of busy P's, block.
	// This is necessary to prevent excessive CPU consumption
	// when GOMAXPROCS>>1 but the program parallelism is low.
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			p2 := allp[enum.position()]
			if _p_ == p2 {
				continue
			}
			if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
				return gp, false
			}

			// Consider stealing timers from p2.
			// This call to checkTimers is the only place where
			// we hold a lock on a different P's timers.
			// Lock contention can be a problem here, so
			// initially avoid grabbing the lock if p2 is running
			// and is not marked for preemption. If p2 is running
			// and not being preempted we assume it will handle its
			// own timers.
			// If we're still looking for work after checking all
			// the P's, then go ahead and steal from an active P.
			if i > 2 || (i > 1 && shouldStealTimers(p2)) {
				tnow, w, ran := checkTimers(p2, now)
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				if ran {
					// Running the timers may have
					// made an arbitrary number of G's
					// ready and added them to this P's
					// local run queue. That invalidates
					// the assumption of runqsteal
					// that is always has room to add
					// stolen G's. So check now if there
					// is a local G to run.
					if gp, inheritTime := runqget(_p_); gp != nil {
						return gp, inheritTime
					}
					ranTimer = true
				}
			}
		}
	}
	if ranTimer {
		// Running a timer may have made some goroutine ready.
		goto top
	}

stop:

	// We have nothing to do. If we're in the GC mark phase, can
	// safely scan and blacken objects, and have work to do, run
	// idle-time marking rather than give up the P.
	if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
		gp := _p_.gcBgMarkWorker.ptr()
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}

	delta := int64(-1)
	if pollUntil != 0 {
		// checkTimers ensures that polluntil > now.
		delta = pollUntil - now
	}

	// wasm only:
	// If a callback returned and no other goroutine is awake,
	// then wake event handler goroutine which pauses execution
	// until a callback was triggered.
	gp, otherReady := beforeIdle(delta)
	if gp != nil {
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}
	if otherReady {
		goto top
	}

	// Before we drop our P, make a snapshot of the allp slice,
	// which can change underfoot once we no longer block
	// safe-points. We don't need to snapshot the contents because
	// everything up to cap(allp) is immutable.
	allpSnapshot := allp

	// return P and block
	lock(&sched.lock)
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
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

	// Delicate dance: thread transitions from spinning to non-spinning state,
	// potentially concurrently with submission of new goroutines. We must
	// drop nmspinning first and then check all per-P queues again (with
	// #StoreLoad memory barrier in between). If we do it the other way around,
	// another thread can submit a goroutine after we've checked all run queues
	// but before we drop nmspinning; as the result nobody will unpark a thread
	// to run the goroutine.
	// If we discover new work below, we need to restore m.spinning as a signal
	// for resetspinning to unpark a new worker thread (because there can be more
	// than one starving goroutine). However, if after discovering new work
	// we also observe no idle Ps, it is OK to just park the current thread:
	// the system is fully loaded so no spinning threads are required.
	// Also see "Worker thread parking/unparking" comment at the top of the file.
	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}
	}

	// check all runqueues once again
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

	// Check for idle-priority GC work again.
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

	// poll network
	if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		atomic.Store64(&sched.pollUntil, uint64(pollUntil))
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		if faketime != 0 {
			// When using fake time, just poll.
			delta = 0
		}
		list := netpoll(delta) // block until new work is available
		atomic.Store64(&sched.pollUntil, 0)
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if faketime != 0 && list.empty() {
			// Using fake time and nothing is ready; stop M.
			// When all M's stop, checkdead will call timejump.
			stopm()
			goto top
		}
		lock(&sched.lock)
		_p_ = pidleget()
		unlock(&sched.lock)
		if _p_ == nil {
			injectglist(&list)
		} else {
			acquirep(_p_)
			if !list.empty() {
				gp := list.pop()
				injectglist(&list)
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false
			}
			if wasSpinning {
				_g_.m.spinning = true
				atomic.Xadd(&sched.nmspinning, 1)
			}
			goto top
		}
	} else if pollUntil != 0 && netpollinited() {
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > pollUntil {
			netpollBreak()
		}
	}
	stopm()
	goto top
}
```

分析上述源码得知查找可用的g的过程如下：

1. 调用 runqget ，尝试从P本地队列中获取g，获取到返回
2. 调用 globrunqget ，尝试从全局队列中获取g，获取到返回
3. 从网络IO轮询器中找到就绪的g，把这个g变为可运行的g
4. 如果不是所有的P都是空闲的，最多四次，随机选一个P，尝试从这P中偷取一些g，获取到返回
5. 上面都找不到g来运行，判断此时P是否处于 `GC mark` 阶段，如果是，那么此时可以安全的扫描和黑化对象和返回 `gcBgMarkWorker` 来运行， `gcBgMarkWorker` 是GC后代标记的goroutine。
6. 再次从全局队列中获取g，获取到返回
7. 再次检查所有的P，有没有可以运行的g
8. 再次检查网络IO轮询器
9. 实在找不到可运行的g了，那就调用 `stopm` 休眠吧

## sysmon

dlv调试：

```shell
[root@583d9a8ec1db p1]# dlv exec ./main
Type 'help' for list of commands.
(dlv) b runtime.sysmon
Breakpoint 1 set at 0x43a773 for runtime.sysmon() /usr/lib/golang/src/runtime/proc.go:4642
(dlv) c
> runtime.sysmon() /usr/lib/golang/src/runtime/proc.go:4642 (hits total:1) (PC: 0x43a773)
Warning: debugging optimized function
  4637: var forcegcperiod int64 = 2 * 60 * 1e9
  4638:
  4639: // Always runs without a P, so write barriers are not allowed.
  4640: //
  4641: //go:nowritebarrierrec
=>4642: func sysmon() {
  4643:         lock(&sched.lock)
  4644:         sched.nmsys++
  4645:         checkdead()
  4646:         unlock(&sched.lock)
  4647:
(dlv) si
> runtime.sysmon() /usr/lib/golang/src/runtime/proc.go:4642 (PC: 0x43a777)
Warning: debugging optimized function
  4637: var forcegcperiod int64 = 2 * 60 * 1e9
  4638:
  4639: // Always runs without a P, so write barriers are not allowed.
  4640: //
  4641: //go:nowritebarrierrec
=>4642: func sysmon() {
  4643:         lock(&sched.lock)
  4644:         sched.nmsys++
  4645:         checkdead()
  4646:         unlock(&sched.lock)
  4647:
```



全部源码：

```go
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)

	lasttrace := int64(0)
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay)
		now := nanotime()
		next, _ := timeSleepUntil()
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
			lock(&sched.lock)
			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
				if next > now {
					atomic.Store(&sched.sysmonwait, 1)
					unlock(&sched.lock)
					// Make wake-up period small enough
					// for the sampling to be correct.
					sleep := forcegcperiod / 2
					if next-now < sleep {
						sleep = next - now
					}
					shouldRelax := sleep >= osRelaxMinNS
					if shouldRelax {
						osRelax(true)
					}
					notetsleep(&sched.sysmonnote, sleep)
					if shouldRelax {
						osRelax(false)
					}
					now = nanotime()
					next, _ = timeSleepUntil()
					lock(&sched.lock)
					atomic.Store(&sched.sysmonwait, 0)
					noteclear(&sched.sysmonnote)
				}
				idle = 0
				delay = 20
			}
			unlock(&sched.lock)
		}
		lock(&sched.sysmonlock)
		{
			// If we spent a long time blocked on sysmonlock
			// then we want to update now and next since it's
			// likely stale.
			now1 := nanotime()
			if now1-now > 50*1000 /* 50µs */ {
				next, _ = timeSleepUntil()
			}
			now = now1
		}

		// trigger libc interceptors if needed
		if *cgo_yield != nil {
			asmcgocall(*cgo_yield, nil)
		}
		// poll network if not polled for more than 10ms
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			list := netpoll(0) // non-blocking - returns list of goroutines
			if !list.empty() {
				// Need to decrement number of idle locked M's
				// (pretending that one more is running) before injectglist.
				// Otherwise it can lead to the following situation:
				// injectglist grabs all P's but before it starts M's to run the P's,
				// another M returns from syscall, finishes running its G,
				// observes that there is no work to do and no other running M's
				// and reports deadlock.
				incidlelocked(-1)
				injectglist(&list)
				incidlelocked(1)
			}
		}
		if next < now {
			// There are timers that should have already run,
			// perhaps because there is an unpreemptible P.
			// Try to start an M to run them.
			startm(nil, false)
		}
		if atomic.Load(&scavenge.sysmonWake) != 0 {
			// Kick the scavenger awake if someone requested it.
			wakeScavenger()
		}
		// retake P's blocked in syscalls
		// and preempt long running G's
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
		// check if we need to force a GC
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			var list gList
			list.push(forcegc.g)
			injectglist(&list)
			unlock(&forcegc.lock)
		}
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
		unlock(&sched.sysmonlock)
	}
}
```

`sysmon` 周期性地检查并retake p， 如果发现p处于这个状态且超过10ms就会强制性收回p，m从cgo和syscall返回后会重新尝试拿p，进入调度循环。
