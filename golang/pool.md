## 深入理解pool
sync.Pool 是用来保存和获取临时对象的

### 结构
```go
type Pool struct {
	noCopy noCopy  // 无法拷贝，保证唯一

	local     unsafe.Pointer // 本地pool，实际类型是[P]poolLocal
	localSize uintptr        // 本地pool的大小

    // get为空的时候，使用new去创建一个object
	New func() interface{}
}
```

```go
type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

type poolLocalInternal struct {
	private interface{}   // 私有数据，不会被其他P获取到
	shared  []interface{} // 共享数据，会被其他P获取
	Mutex                 // 锁
}

```

### 使用

#### put

```go
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
    }
    // 获取poolLocal
    l := p.pin()
    // 如果私有数据没有，则将x复制给private
	if l.private == nil {
		l.private = x
		x = nil
    }
    // 解除抢占
	runtime_procUnpin()
	if x != nil {
        // 说明private不为nil，将x直接追加到shared里面
        l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
	if race.Enabled {
		race.Enable()
	}
}
```

#### pin
pin的功能是绑定goroutine到固定P上，并且不允许抢占, 这里利用m.locks 标记
```go
func (p *Pool) pin() *poolLocal {
    // 返回绑定后P的id, 不允许抢占， 相关函数在runtime/proc.go中sync_runtime_procPin
	pid := runtime_procPin()
    // 本地pool的大小
    s := atomic.LoadUintptr(&p.localSize)
    // 本地的pool
    l := p.local
    // 当前的P有pool，直接返回poolLocal        
	if uintptr(pid) < s {
		return indexLocal(l, pid)
    }
    // 需要重新分配，因为GOMAXPROCS在GC的时候改变了
	return p.pinSlow()
}
```

#### pinSlow
再次判断获取下poolLocal
```go
func (p *Pool) pinSlow() *poolLocal {
    // 解除抢占
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
    // 当我们在绑定的时候，poolCleanup不会被调用
	s := p.localSize
    l := p.local
    // 再次判断下
	if uintptr(pid) < s {
		return indexLocal(l, pid)
    }
    // 走到这里，分两种情况：
    // 1. Pool首次调用
    // 2. GOMAXPROCS 发生改变了

    // p.local为空，说明是新增的Pool，append到allPools
	if p.local == nil {
		allPools = append(allPools, p)
	}

    // 如果GOMAXPROCS在GC的时候改变了，我们会重新分配, GOMAXPROCS(0)会返回当前的Procs
	size := runtime.GOMAXPROCS(0)
    local := make([]poolLocal, size)
    // 存储local和localsize
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0]))
	atomic.StoreUintptr(&p.localSize, uintptr(size))     
	return &local[pid]
}
```

#### poolCleanup
清除pool，这里比较暴力，直接全部清除
```go
func poolCleanup() {
    // poolCleanup执行的时候会在GC开始的时候STW
    // 将所有内容清空主要有两个原因：
    // 1. 防止错误的保留整个Pools
    // 2. 如果GC和l.shared Put/Get同时发生，那么就会保留整个Pool，下个内存的消耗将增加一倍
	for i, p := range allPools {
        allPools[i] = nil
        // 遍历所有local pool
		for i := 0; i < int(p.localSize); i++ {
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				l.shared[j] = nil
			}
			l.shared = nil
		}
		p.local = nil
		p.localSize = 0
	}
	allPools = []*Pool{}
}
```


### 总结
sync.Pool还是挺简单的，Pool内存清除是发生在GC的时候