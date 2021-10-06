---
title: golang运行时队列操作函数源码分析
date: 2021-10-06 09:32:36
index_img: /img/golang/go.jpg
banner_img:
categories: 
- golang
tags:
- golang
- runtime
---

> 环境：
> 
> CentOS Linux release 8.4.2105
> 
> go1.15.14
> 
> dlv1.5.0
> 
> 被调试的源码内容：

```golang
package main

func main() {
	println("Hello World!")
}
```

> 快速搭建调试环境:

```Dockerfile
    FROM centos
    RUN yum install golang -y \
    && yum install dlv -y \
    && yum install binutils -y \
    && yum install vim -y \
    && yum install gdb -y
```

# golang运行时概述

Go的调度流程本质上是生产-消费流程

1. 生产端生成goruntine放入队列
2. 消费端通过与M绑定的P获取goroutine
3. M循环调度执行 runtime.schedule->runtime.execute->runtime.gogo->runtime.goexit

为了缓解一级队列中生产消费模型的压力，Go采用三级队列：

1. P中的runnext队列, 该队列只保存一个goroutine
2. local run queue 该队列最大保存256个goroutine
3. global run queue 该队列以链表的形式保存goroutine

操作运行时goroutine队列的函数主要有：

1. runqput
2. runqget
3. globrunqput
4. globrunqget

> 为了方便描述一下内容所有的g代指goroutine

## runqput

使用dlv调试

```sh
[root@583d9a8ec1db p1]# dlv exec ./main
Type 'help' for list of commands.
(dlv) b runtime.runqput
Breakpoint 1 set at 0x43c073 for runtime.runqput() /usr/lib/golang/src/runtime/proc.go:5153
(dlv) c
> runtime.runqput() /usr/lib/golang/src/runtime/proc.go:5153 (hits total:1) (PC: 0x43c073)
Warning: debugging optimized function
  5148: // runqput tries to put g on the local runnable queue.
  5149: // If next is false, runqput adds g to the tail of the runnable queue.
  5150: // If next is true, runqput puts g in the _p_.runnext slot.
  5151: // If the run queue is full, runnext puts g on the global queue.
  5152: // Executed only by the owner P.
=>5153: func runqput(_p_ *p, gp *g, next bool) {
  5154:         if randomizeScheduler && next && fastrand()%2 == 0 {
  5155:                 next = false
  5156:         }
  5157:
  5158:         if next {
```

runqput 全部源码内容：

```go
    func runqput(_p_ *p, gp *g, next bool) {
        if randomizeScheduler && next && fastrand()%2 == 0 {
            next = false
        }

        if next {
        retryNext:
            oldnext := _p_.runnext
            if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
                goto retryNext
            }
            if oldnext == 0 {
                return
            }
            // Kick the old runnext out to the regular run queue.
            gp = oldnext.ptr()
        }

    retry:
        h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
        t := _p_.runqtail
        if t-h < uint32(len(_p_.runq)) {
            _p_.runq[t%uint32(len(_p_.runq))].set(gp)
            atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
            return
        }
        if runqputslow(_p_, gp, h, t) {
            return
        }
        // the queue is not full, now the put above must succeed
        goto retry
    }
```

从该函数可以看出goroutine放入队列主要有一下一些逻辑

1. 当入参next为false时:

   1) 尝试将g放入当前P的runq队列，我们称为本地队列
   2) 当本地队列已经满了的时候调用  runqputslow 函数
   3) runqputslow函数执行批量的将本地队列的一半大小和当前的g一起移动到全局队列

2. 当入参next为true时:

    1) 尝试将g放入当前P的runnext中,将原先存在的g从runnext中取出
    2) 尝试将取出的g放入当前P的runq队列，我们称为本地队列
    3) 当本地队列已经满了的时候调用  runqputslow 函数
    4) runqputslow函数执行批量的将本地队列的一半大小和取出的g一起移动到全局队列

## runqget

dlv 调试runqget

```sh
(dlv) b runtime.runqget
Breakpoint 2 set at 0x43c4a0 for runtime.runqget() /usr/lib/golang/src/runtime/proc.go:5265
(dlv) c
> runtime.runqget() /usr/lib/golang/src/runtime/proc.go:5265 (hits total:1) (PC: 0x43c4a0)
Warning: debugging optimized function
  5260:
  5261: // Get g from local runnable queue.
  5262: // If inheritTime is true, gp should inherit the remaining time in the
  5263: // current time slice. Otherwise, it should start a new time slice.
  5264: // Executed only by the owner P.
=>5265: func runqget(_p_ *p) (gp *g, inheritTime bool) {
  5266:         // If there's a runnext, it's the next G to run.
  5267:         for {
  5268:                 next := _p_.runnext
  5269:                 if next == 0 {
  5270:                         break
```

通过调试信息找到源代码：

```go
    func runqget(_p_ *p) (gp *g, inheritTime bool) {
        // If there's a runnext, it's the next G to run.
        for {
            next := _p_.runnext
            if next == 0 {
                break
            }
            if _p_.runnext.cas(next, 0) {
                return next.ptr(), true
            }
        }

        for {
            h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
            t := _p_.runqtail
            if t == h {
                return nil, false
            }
            gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
            if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
                return gp, false
            }
        }
    }
```

从源码中可以看出 runqget主要操作P的本地队列， 优先获取runnext之后再获取runq中的头个

## globalrunqput

dlv 调试

```sh
[root@583d9a8ec1db p1]# dlv exec ./main
Type 'help' for list of commands.
(dlv) b runtime.globrunqput
Breakpoint 1 set at 0x435b46,0x43670e,0x437bb0,0x4396fc,0x453ca6,0x454814 for runtime.injectglist() /usr/lib/golang/src/runtime/proc.go:5044
(dlv) c
> runtime.goschedImpl() /usr/lib/golang/src/runtime/proc.go:5043 (hits total:1) (PC: 0x43670e)
Warning: debugging optimized function
  5038: // Put gp on the global runnable queue.
  5039: // Sched must be locked.
  5040: // May run during STW, so write barriers are not allowed.
  5041: //go:nowritebarrierrec
  5042: func globrunqput(gp *g) {
=>5043:         sched.runq.pushBack(gp)
  5044:         sched.runqsize++
  5045: }
  5046:
  5047: // Put gp at the head of the global runnable queue.
  5048: // Sched must be locked.
```

该函数逻辑比较简单 只是将g添加到全局队列的链表中

## globrunqget

dlv 调试

```sh
[root@583d9a8ec1db p1]# dlv exec ./main
Type 'help' for list of commands.
(dlv) b runtime.globrunqget
Breakpoint 1 set at 0x43be33 for runtime.globrunqget() /usr/lib/golang/src/runtime/proc.go:5067
(dlv) c
> runtime.globrunqget() /usr/lib/golang/src/runtime/proc.go:5067 (hits total:1) (PC: 0x43be33)
Warning: debugging optimized function
  5062:         *batch = gQueue{}
  5063: }
  5064:
  5065: // Try get a batch of G's from the global runnable queue.
  5066: // Sched must be locked.
=>5067: func globrunqget(_p_ *p, max int32) *g {
  5068:         if sched.runqsize == 0 {
  5069:                 return nil
  5070:         }
  5071:
  5072:         n := sched.runqsize/gomaxprocs + 1
```

找到对应源码为:

```go
    func globrunqget(_p_ *p, max int32) *g {
        if sched.runqsize == 0 {
            return nil
        }

        n := sched.runqsize/gomaxprocs + 1
        if n > sched.runqsize {
            n = sched.runqsize
        }
        if max > 0 && n > max {
            n = max
        }
        if n > int32(len(_p_.runq))/2 {
            n = int32(len(_p_.runq)) / 2
        }

        sched.runqsize -= n

        gp := sched.runq.pop()
        n--
        for ; n > 0; n-- {
            gp1 := sched.runq.pop()
            runqput(_p_, gp1, false)
        }
        return gp
    }
```

分析源码发现该函数主要逻辑：

1. 批量取出部分全局队列的g，取出数量= 全局总量/核心数+1
2. 将 全局总量/核心数 的g加入到当前P的本地队列
3. 返回一个g

### 总结

获取： g的获取优先从runnext 然后本地runq 最终获取不到去全局获取
添加： 优先放入runnext 然后本地队列 本地队列满了批量放入全局队列
