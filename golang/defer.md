## Defer 函数

### defer
defer函数的参数在声明之前，其实已经确定了值


### defer  结构体
```go
	type _defer struct {
		siz     int32   // 参数的大小
		started bool    // 是否执行过了
		sp      uintptr // 函数栈指针
		pc      uintptr 
		fn      *funcval 
		_panic  *_panic // defer中的panic
		link    *_defer // 通过链表的形势存储_defer
	}
```

### deferproc
```go

func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
	if getg().m.curg != getg() {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}

	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()
	// 新建一个defer
	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	// 设置defer信息
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		// 如果是指针，_defer后面内存存储参数地址信息
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		// 如果不是指针类型，把参数拷贝到_defer后面
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	// deferproc returns 0 normally.
	// a deferred func that stops a panic
	// makes the deferproc return 1.
	// the code the compiler generates always
	// checks the return value and jumps to the
	// end of the function if deferproc returns != 0.
	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}
```


### newdefer
newdefer 是获取一个_defer对象，并把它加入到链表头部
```go
func newdefer(siz int32) *_defer {
	var d *_defer
	// 根据size通过deferclass得出应该分配的sizeclass，系统会预先分配好一些sizeclass，然后根据对应的sz，去直接读取缓存
	sc := deferclass(uintptr(siz))
	gp := getg()
	// 在当前p的范围内，去p上找
	if sc < uintptr(len(p{}.deferpool)) {
		pp := gp.m.p.ptr()
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			// 当前p上对应sc的缓存为0，则从sched上获取一批
			systemstack(func() {
				lock(&sched.deferlock)
				for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
					d := sched.deferpool[sc]
					sched.deferpool[sc] = d.link
					d.link = nil
					pp.deferpool[sc] = append(pp.deferpool[sc], d)
				}
				unlock(&sched.deferlock)
			})
		}
		// 从sched获取到了，并且不为空，则分配
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
	}
	if d == nil {
		// 如果还没找到，则直接分配
		systemstack(func() {
			total := roundupsize(totaldefersize(uintptr(siz)))
			d = (*_defer)(mallocgc(total, deferType, true))
		})
		if debugCachedWork {
			// Duplicate the tail below so if there's a
			// crash in checkPut we can tell if d was just
			// allocated or came from the pool.
			d.siz = siz
			d.link = gp._defer
			gp._defer = d
			return d
		}
	}
	d.siz = siz
	// 插入链表头部
	d.link = gp._defer
	gp._defer = d
	return d
}
```

### deferreturn
```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	// 获取最后一个声明的defer
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}

	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
	// defer用完了就盛放
	gp._defer = d.link
	// 释放defer
	freedefer(d)
	// 跳转执行其他defer
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

### freedefer
释放defer
```go
func freedefer(d *_defer) {

	if d._panic != nil {
		freedeferpanic()
	}
	if d.fn != nil {
		freedeferfn()
	}
	// 判断sizeclass
	sc := deferclass(uintptr(d.siz))
	// 超过当前deferpool的范围，则直接返回
	if sc >= uintptr(len(p{}.deferpool)) {
		return
	}
	pp := getg().m.p.ptr()
	// 本地容量已经满了
	if len(pp.deferpool[sc]) == cap(pp.deferpool[sc]) {
		// 转移一半到sched
		systemstack(func() {
			var first, last *_defer
			for len(pp.deferpool[sc]) > cap(pp.deferpool[sc])/2 {
				n := len(pp.deferpool[sc])
				d := pp.deferpool[sc][n-1]
				pp.deferpool[sc][n-1] = nil
				pp.deferpool[sc] = pp.deferpool[sc][:n-1]
				if first == nil {
					first = d
				} else {
					last.link = d
				}
				last = d
			}
			lock(&sched.deferlock)
			last.link = sched.deferpool[sc]
			sched.deferpool[sc] = first
			unlock(&sched.deferlock)
		})
	}

	// 清空defer数据
	d.siz = 0
	d.started = false
	d.sp = 0
	d.pc = 0
	d.link = nil

	pp.deferpool[sc] = append(pp.deferpool[sc], d)
}

```


### 说明
从上面可以看出，defer函数的参数在defer语句的时候已经定义下来了，从下面几个例子我们着重说明

#### Example1

```go
func demo1() int {
  	x := 1
  	defer func() {
		x++ 
	}()
	return x 
}
```
先看第一个例子，这个应该输出什么？ 答案是1，为什么会输出1，我们试着转换一下,是不是就很清晰了，最后返回的起始一个已经被赋值后的变量tmp，跟x就没关系了

```go
func demo1() int {
	x := 1
	tmp := x
	x++
	return tmp
}
```

#### Example2

```go
func demo2() (x int) {
	x=1
  	defer func() {
    	x++
	}()
	return 
}
```
第二个例子输出什么呢？ 如果你能马上答出2，那么说明你已经完全理解第一个例子了，那么我们还是翻译一下, 如果这还不明白，请面壁思过下

```go
func demo2() (x int) {
	x=1
  	x++
	return 
}
```

#### Example3

```go
func demo3() (y int) {
	x := 1
  	defer func() {
    	x++
	}()
	return x 
}
```
第三个例子呢， 输出是1，其实第三个和第一个是一种情况，只不过有一些误导性，再看下第一个例子就会明白了