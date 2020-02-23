## 深入理解time

### 数据结构

#### timer

```go
type timer struct {
    // timer所在的bucket
    tb *timersBucket // the bucket the timer lives in
    // 堆排序的索引
	i  int           // heap index

    // Timer会由在when或者wen+period达到时唤醒，然后执行f(arg, now), f应该是个非阻塞的函数
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
}
```

##### assignBucket
找到timer对应timersBucket
```go
func (t *timer) assignBucket() *timersBucket {
	id := uint8(getg().m.p.ptr().id) % timersLen
	t.tb = &timers[id].timersBucket
	return t.tb
}
```

#### timers
timers是个长度为64的数组，timers包含了每个P对应的时间堆，当GOMAXPROCS>timersLen，则会有一些
```go
var timers [timersLen]struct {
	timersBucket   // 64

	// The padding should eliminate false sharing
	// between timersBucket values.
	pad [cpu.CacheLinePadSize - unsafe.Sizeof(timersBucket{})%cpu.CacheLinePadSize]byte
}
```

#### timersBucket

```go
type timersBucket struct {
	lock         mutex
	gp           *g
	created      bool
	sleeping     bool
	rescheduling bool
	sleepUntil   int64
	waitnote     note
	t            []*timer
}
```

#### sleep
sleep会将当前的goroutine休眠直到ns 纳秒
```go
func timeSleep(ns int64) {
	// 小于或等于0，直接返回
	if ns <= 0 {
		return
	}
	// 获取当前g
	gp := getg()
	t := gp.timer
	if t == nil {
		t = new(timer)
		gp.timer = t
	}
	*t = timer{}
	// 唤醒时间
	t.when = nanotime() + ns
	// 初始化唤醒函数
	t.f = goroutineReady
	t.arg = gp
	tb := t.assignBucket()
	lock(&tb.lock)
	if !tb.addtimerLocked(t) {
		unlock(&tb.lock)
		badTimer()
	}
	goparkunlock(&tb.lock, waitReasonSleep, traceEvGoSleep, 2)
}
```

#### addtimerLocked
addtimerLocked 将timer加入到堆中
```go
func (tb *timersBucket) addtimerLocked(t *timer) bool {
	// when must never be negative; otherwise timerproc will overflow
	// during its delta calculation and never expire other runtime timers.
	// when永远不能是负数，否则会导致timeproc溢出和其他运行中的timers永远不会过期
	if t.when < 0 {
		t.when = 1<<63 - 1
	}
	t.i = len(tb.t)
	// 将timer加入到最后
	tb.t = append(tb.t, t)
	// 重新排序，从上到下
	if !siftupTimer(tb.t, t.i) {
		return false
	}
	if t.i == 0 {
		// siftup moved to top: new earliest deadline.
		// 已经到头部，创建更早的触发线
		if tb.sleeping && tb.sleepUntil > t.when {
			// 当前tb在休眠中，并休眠触发时间大于新加入timer触发时间，尝试修改唤醒时间
			tb.sleeping = false
			notewakeup(&tb.waitnote)
		}
		// 退出
		if tb.rescheduling {
			tb.rescheduling = false
			goready(tb.gp, 0)
		}
		// 创建timeproc
		if !tb.created {
			tb.created = true
			go timerproc(tb)
		}
	}
	return true
}
```

#### siftupTimer
[]*timer是个按时间从早到晚的四叉树
```go
func siftupTimer(t []*timer, i int) bool {
	// 正常的情况i = len(t) + 1
	if i >= len(t) {
		return false
	}
	// i是最后的节点
	when := t[i].when
	tmp := t[i]
	for i > 0 {
		p := (i - 1) / 4 // parent
		// 大于或等于父节点的触发时间，跳出循环
		if when >= t[p].when {
			break
		}
		// 新加入节点的触发时间小于父节点，和父节点调换
		t[i] = t[p]
		t[i].i = i
		i = p
	}
	// 有顺序调换
	if tmp != t[i] {
		t[i] = tmp
		t[i].i = i
	}
	return true
}
```


#### timerproc
timerproc是个以时间驱动的事件，直到堆中的事件被触发，如果有一个更早的事件，会直接唤醒timeproc
```go
func timerproc(tb *timersBucket) {
	// 获取当前绑定的g
	tb.gp = getg()
	for {
		// 加锁
		lock(&tb.lock)
		tb.sleeping = false
		now := nanotime()
		delta := int64(-1)
		for {
			// 当前没有事件
			if len(tb.t) == 0 {
				delta = -1
				break
			}
			t := tb.t[0]
			delta = t.when - now
			// 还没有达到最早的时间，退出
			if delta > 0 {
				break
			}
			ok := true
			if t.period > 0 {
				// leave in heap but adjust next time to fire
				// 从heap中去除，并重新更新触发时间
				t.when += t.period * (1 + -delta/t.period)
				if !siftdownTimer(tb.t, 0) {
					ok = false
				}
			} else {
				// 从tb中删除
				last := len(tb.t) - 1
				if last > 0 {
					tb.t[0] = tb.t[last]
					tb.t[0].i = 0
				}
				tb.t[last] = nil
				tb.t = tb.t[:last]
				// 重新排序
				if last > 0 {
					if !siftdownTimer(tb.t, 0) {
						ok = false
					}
				}
				// 标记当前已被删除
				t.i = -1 // mark as removed
			}
			f := t.f
			arg := t.arg
			seq := t.seq
			unlock(&tb.lock)
			if !ok {
				badTimer()
			}
			if raceenabled {
				raceacquire(unsafe.Pointer(t))
			}
			// 调用触发函数
			f(arg, seq)
			// 加锁，继续下一次判断
			lock(&tb.lock)
		}
		// 没有事件触发，将当前goroutine休眠
		if delta < 0 || faketime > 0 {
			// No timers left - put goroutine to sleep.
			tb.rescheduling = true
			goparkunlock(&tb.lock, waitReasonTimerGoroutineIdle, traceEvGoBlock, 1)
			continue
		}
		// 至少有一个timer在pending，休眠直接到下次唤醒
		tb.sleeping = true
		tb.sleepUntil = now + delta
		noteclear(&tb.waitnote)
		unlock(&tb.lock)
		notetsleepg(&tb.waitnote, delta)
	}
}
```


#### siftdownTimer
从上到下进行排序
```go
func siftdownTimer(t []*timer, i int) bool {
	n := len(t)
	if i >= n {
		return false
	}
	// i = 0
	when := t[i].when
	tmp := t[i]
	for {
		c := i*4 + 1
		c3 := c + 2 
		if c >= n {
			break
		}
		w := t[c].when
		if c+1 < n && t[c+1].when < w {
			w = t[c+1].when
			c++
		}
		if c3 < n {
			w3 := t[c3].when
			if c3+1 < n && t[c3+1].when < w3 {
				w3 = t[c3+1].when
				c3++
			}
			if w3 < w {
				w = w3
				c = c3
			}
		}
		// 找到位置
		if w >= when {
			break
		}
		// 重新更新
		t[i] = t[c]
		t[i].i = i
		i = c
	}
	if tmp != t[i] {
		t[i] = tmp
		t[i].i = i
	}
	return true
}
```


#### deltimer
从堆中删除timer，不用更新timerproc，触发了也没什么大不了（官方文档这么写的）
```go
func deltimer(t *timer) bool {
	if t.tb == nil {
		// t.tb can be nil if the user created a timer
		// directly, without invoking startTimer e.g
		//    time.Ticker{C: c}
		// In this case, return early without any deletion.
		// See Issue 21874.
		// t.tb可以为空
		return false
	}

	tb := t.tb

	lock(&tb.lock)
	removed, ok := tb.deltimerLocked(t)
	unlock(&tb.lock)
	if !ok {
		badTimer()
	}
	return removed
}

```

#### deltimerLocked
```go
func (tb *timersBucket) deltimerLocked(t *timer) (removed, ok bool) {
	// t may not be registered anymore and may have
	// a bogus i (typically 0, if generated by Go).
	// Verify it before proceeding.
	i := t.i
	last := len(tb.t) - 1
	if i < 0 || i > last || tb.t[i] != t {
		return false, true
	}
	// 如果删除的不是尾节点
	if i != last {
		tb.t[i] = tb.t[last]
		tb.t[i].i = i
	}
	tb.t[last] = nil
	tb.t = tb.t[:last]
	ok = true
	if i != last {
		// 从下到上排序下
		if !siftupTimer(tb.t, i) {
			ok = false
		}
		// 从上到下排序下
		if !siftdownTimer(tb.t, i) {
			ok = false
		}
	}
	return true, ok
}

```