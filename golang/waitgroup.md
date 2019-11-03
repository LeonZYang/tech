## waitgroup
waitgroup主要是用来等待一些goroutine完成

### 数据结构
```go
type WaitGroup struct {
	noCopy noCopy   // 不允许拷贝，用go vet检测

    // 高32位存储counter数量，低32位存储waiter数量，最后4个字节存储sema
	state1 [3]uint32
}
```

### 主要方法

#### Add
```go
func (wg *WaitGroup) Add(delta int) {
    // 获取statep和sema
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
    }
    // goroutine+delta，delta可能是负数
	state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32)
    // 获取waiters
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
		// The first increment must be synchronized with Wait.
		// Need to model this as a read, because there can be
		// several concurrent wg.counter transitions from 0.
		race.Read(unsafe.Pointer(semap))
    }
    // add后的counter不能为负数
	if v < 0 {
		panic("sync: negative WaitGroup counter")
    }
    // 此时wait已经执行(waiter>0)，不能再执行Add
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 没有waiter 直接return
	if v > 0 || w == 0 {
		return
	}

    // 到这里v=0， 并w>0，这里状态又不一致，说明又有别的add改变了状态
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// 重置waiters为0
    *statep = 0
    // 开始批量释放信号量
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false)
	}
}
```

#### Done
Done就很简单，相当于Add(-1)
```go
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

#### Wait
```go
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
        state := atomic.LoadUint64(statep)
        // 获取counter和waiter的值
		v := int32(state >> 32)
		w := uint32(state)
		if v == 0 {
			// counter为0，不需要等待
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
        // 增加waiter的count
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				race.Write(unsafe.Pointer(semap))
            }
            // 阻塞的获取sema
            runtime_Semacquire(semap)
            // Add都结束了，但是又有Add执行了
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
```