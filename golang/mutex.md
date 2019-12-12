## Mutex

### 常量
```go
	mutexLocked = 1 << iota // 是否被锁定
	mutexWoken              // 是否被唤醒
	mutexStarving           // 是否处理饥饿模式
	mutexWaiterShift = iota // 等待的goroutine

	// mutex有两种模式：正常模式和饥饿模式
	// 正常模式下，所有的等待者都会按照FIFO的顺序，
	// 如果有超过1个等待者获取锁失败，那么就会进入饥饿模式

	// 在饥饿模式下，所有新加入的等待者不会尝试去获取锁，而是直接加入等待者队列队尾
	// 下列两种情况会从饥饿模式变为正常模式
	// 1. 这个waiter已经是队尾
	// 2. 获取锁低于1ms

	// 饥饿模式的阈值 1ms
	starvationThresholdNs = 1e6 
```

### 数据结构
```go
type Mutex struct {
	state int32    // 状态
	sema  uint32   // 信号量
}
```

#### state
state是个32位的二进制
xxxxxxxxxxxxxxxxxxxxxxxxxxxxx|x|y|z

x 代表：mutexWaiterShift
y 代表：mutexWoken
z 代表：mutexLocked
剩下的代表：mutexWaiterShift

### Lock and Unlock

#### CAS算法
CAS(Compare and swap)是一种乐观锁，在go里面是以CompareAndSwap为前缀的函数
```go
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
```
* 首先会判断addr的值是否和old相等
* 相等则将new替换掉old当前值，否则就会忽略

#### Lock
```go
func (m *Mutex) Lock() {
    // 如果锁定成功，则直接返回
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}

    // 等待时间
	var waitStartTime int64
    // 饥饿模式
	starving := false
    awoke := false
    // 迭代次数
    iter := 0
    // 状态
	old := m.state
	for {
		// 被锁定，并且没有处在饥饿模式下，可以自旋
		// runtime_canSpin => runtime.sync_runtime_canSpin 如果iter>=4或者只有1个核，则不能发自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 没有处在mutexWoken并且当前有阻塞的goroutines，则开始唤醒
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			// 开始自旋
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
        // 如果没有处在饥饿模式下，则直接设置锁定状态
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		// 如果当前被锁定并且处于饥饿模式，则直接将state对应的偏移量加1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		// 如果当前是锁定状态，并且是触发饥饿模式逻辑，则进入姐模式
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// 状态不一致
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// goroutine已经被唤醒，我们需要重置flag
			new &^= mutexWoken
		}
		// 获取锁成功
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				// 已经被CAS锁定
				break
			}

			// 如果waitStartTime 不为0，那么说明之前已经有等待的，那么按LIFO
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo)
			// 获取锁的失败的时间，如果大于1ms，则认为是饥饿状态
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// 如果goroutine已经被唤醒，并且处于饥饿模式，不一致状态
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				
				if !starving || old>>mutexWaiterShift == 1 {
					// 如果不处于饥饿阈值或者当前是只有1个blocked goroutine，那么退出饥饿模式
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```


#### Unlock

```go
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	new := atomic.AddInt32(&m.state, -mutexLocked)
	// 如果当前状态是unlock状态，则panic
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}

	if new&mutexStarving == 0 {
		// 如果不是饥饿模式
		old := new
		// 循环去
		for {
			// 如果没有等待者或者当前处在锁定、唤醒（被其他goroutine解锁）和饥饿状态，则直接返回
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}

			// 等待的waiters 减1， 并且变为唤醒模式
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		// 如果是饥饿模式
		runtime_Semrelease(&m.sema, true)
	}
}
```

### 读写锁
上面我们介绍了互斥锁，接下来我们说下读写锁

#### 数据结构

```go
type RWMutex struct {
	w           Mutex 
	writerSem   uint32 // 操作写锁的信号量
	readerSem   uint32 // 操作读锁的信号量
	readerCount int32  // 需要加读锁的数量
	readerWait  int32  // 已经获取读锁的数量
}
```

rwmutexMaxReaders = 1 << 30 相当于2^30次，是个很大的数字，用来操作读统计，下面会具体说明

#### Rlock
```go
func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// 如果readerCount+1小于0，那说明有个写锁在操作
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// 等待写锁完成，
		runtime_SemacquireMutex(&rw.readerSem, false)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}
```

runtime_SemacquireMutex 会block直到readerSem > 0

#### Lock

```go
func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}

	// 首先，加写锁，保证其他不会有其他写锁同时操作
	rw.w.Lock()
	// readerCount-rwmutexMaxReaders 会得到一个很大的负数，这样加读锁的时候atomic.AddInt32(&rw.readerCount, 1)会小于0
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// 等待readerWait的读锁释放
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
```

#### RUnlock

```go
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	// 将readerCounter 减1
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// 当RUnlock一个正在Lock的或者当前没有读锁
		if r+1 == 0 || r+1 == -rwmutexMaxReaders {
			race.Enable()
			throw("sync: RUnlock of unlocked RWMutex")
		}

		// 如果当前有写锁正在等待
		if atomic.AddInt32(&rw.readerWait, -1) == 0 {
			// 通知writerSem 可以获取写锁了
			runtime_Semrelease(&rw.writerSem, false)
		}
	}
	if race.Enabled {
		race.Enable()
	}
}
```

#### Unlock

```go
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// 恢复readerCounte，告诉其他读锁，现在没有写锁了
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	// 说明现在没有写锁
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}

	// 开始循环通知各个读锁
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
```


#### 总结
从读写锁总结出如下规律
1. 读锁需要等写锁解锁后才能锁定
2. 写锁需要等读锁解锁后锁定
3. 写锁之间有互斥
4. 读锁之间没有关系
