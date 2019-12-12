## 深入理解sync.Map

### 数据结构

#### Map
```go
type Map struct {
	mu Mutex  // 锁

	read atomic.Value // 优先读取map

	dirty map[interface{}]*entry  // dirty当前最新的map，提供读写

	misses int // 在read中读取不到读取dirty map的次数，当misses=len(dirty)，则将dirty拷贝到read
}
```

#### readOnly
```go
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // dirty中存在但是不在m，则为真
}
```

#### entry
```go
type entry struct {
    // 如果p为nil， entry已被删除并且m.dirty=nil
    // 如果p==expunged， 表示entry已被删除，并且dirty不为nil，并且entry不存在dirty中
    // 其他entry是个正常值
	p unsafe.Pointer // *interface{}
}
```

#### expunged
expunged是个空指针表示dirty中已删除

```go
var expunged = unsafe.Pointer(new(interface{}))
```

### 使用

#### Load
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	// 第一次从read中获取数据
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		// 数据不存在并且dirty不为空
		m.mu.Lock()
		// 避免因为加锁过程中（阻塞），dirty被提升为read
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 不管在dirty中有没有找到，都记录为miss
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}
```

##### missLocked
当misses大于等于m.dirty的长度，则将dirty提升为read
```go
func (m *Map) missLocked() {
	m.misses++
	if m.misses < len(m.dirty) {
		return
    }
	m.read.Store(readOnly{m: m.dirty})
	m.dirty = nil
	m.misses = 0
}
```

##### load
```go
func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
	// 如果p已经删除
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}
```

#### Store
```go
func (m *Map) Store(key, value interface{}) {
	// 第一次获取readOnly
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		// key存在并且没有被删除
		return
	}

	m.mu.Lock()
	// 再次获取下readOnly
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 确保未被标记成删除
		if e.unexpungeLocked() {
			// 该条目之前被删除了，这意味着有非nil的dirty map，并且entry不在dirty map中
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		// dirty map中存在
		e.storeLocked(&value)
	} else {
		if !read.amended {
			// dirty map为空，增加到dirty map中
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}
```

storeLocked原子存储数据（必须确定entry没有删除）

##### dirtyLocked
dirty为nil，将read中的数据拷贝到dirty中
```go
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		// nil和expunged不拷贝到dirty
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}

```


#### LoadOrStore
```go
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
	// 不加锁读取read
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 如果entry成员在，尝试获取值
		actual, loaded, ok := e.tryLoadOrStore(value)
		if ok {
			// entry存在或者赋值成功，ok为true
			return actual, loaded
		}
	}

	m.mu.Lock()
	// 第二次获取read
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		// 如果之前标记为expunged，写入到dirty
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		actual, loaded, _ = e.tryLoadOrStore(value)
	} else if e, ok := m.dirty[key]; ok {
		actual, loaded, _ = e.tryLoadOrStore(value)
		m.missLocked()
	} else {
		if !read.amended {
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
		actual, loaded = value, false
	}
	m.mu.Unlock()

	return actual, loaded
}
```


##### tryLoadOrStore
```go
func (e *entry) tryLoadOrStore(i interface{}) (actual interface{}, loaded, ok bool) {
	p := atomic.LoadPointer(&e.p)
	// 已经删除
	if p == expunged {
		return nil, false, false
	}
	// entry存在
	if p != nil {
		return *(*interface{})(p), true, true
	}

	ic := i
	for {
		// entry为空，则赋值
		if atomic.CompareAndSwapPointer(&e.p, nil, unsafe.Pointer(&ic)) {
			return i, false, true
		}
		p = atomic.LoadPointer(&e.p)
		if p == expunged {
			return nil, false, false
		}
		if p != nil {
			return *(*interface{})(p), true, true
		}
	}
}
```

#### Delete

```go
func (m *Map) Delete(key interface{}) {
    // 第一次获取
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// 第二次获取
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			// read中不存在，并且dirty不为nil，不管dirty中存不存在，都执行删除
			delete(m.dirty, key)
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete()
	}
}

func (e *entry) delete() (hadValue bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		// 如果已经删除了
		if p == nil || p == expunged {
			return false
		}
		// 删除很简单，直接将entry设置为nil，这样的好处是能根据此判断是否删除
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}
```


#### Range
```go
func (m *Map) Range(f func(key, value interface{}) bool) {
	// 迭代的获取所有元素
	read, _ := m.read.Load().(readOnly)
	if read.amended {
		// dirty不为nil，dirty有些keys 不在read
		m.mu.Lock()
		// 第二次检测
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			// 如果dirty依然不为nil，则直接将diry提升为read
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```

sync.Map 没有实现Len方法，可以用Range配合计数实现，这里就不多说了

### 总结
sync.Map 通过read和dirty两个map控制，将读取任务主要集中在read中，在源码中dirty和read需要互相拷贝，如果数据量比较大并且有很大的读写，sync.Map就不是很适合了。