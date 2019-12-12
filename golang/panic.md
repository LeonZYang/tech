## 深入理解Panic

### 数据结构
```go
//go:notinheap
type _panic struct {
	argp      unsafe.Pointer // 对应panic函数的指针
	arg       interface{}    // panic的参数
	link      *_panic        // 链接上一个panic，用链表存储
	recovered bool           // 是否已被recovery
	aborted   bool           // 是否已中止
}
```

panic具体调用的函数，我们可以用个例子写出
```go
package main

func main(){
    panic("error")
}
```

```bash
# go tool compile -S main.go

    ...
    0x0028 00040 (panic_demo.go:4)	LEAQ	"".statictmp_0(SB), AX
    0x002f 00047 (panic_demo.go:4)	PCDATA	$2, $0
    0x002f 00047 (panic_demo.go:4)	MOVQ	AX, 8(SP)
    0x0034 00052 (panic_demo.go:4)	CALL	runtime.gopanic(SB)
    ...
```
可以看到是调用的gopanic函数，那我们看下gopanic函数做了什么

```go
func gopanic(e interface{}) {
	gp := getg()     // 获取当前goroutine的指针
	//...

	var p _panic           // 定义了个_panic的结构体
	p.arg = e      
	p.link = gp._panic  
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	atomic.Xadd(&runningPanicDefers, 1)

	// 处理所有的defer 
	for {
		d := gp._defer       // 获取defer，如果没有则退出
		if d == nil {
			break
		}

		// 如果defer 被上一个panic或者goexit 触发，那么这里会触发一个新的panic
		// 之前的painc的会标记为中止
		if d.started {
			if d._panic != nil {
				d._panic.aborted = true
			}
			d._panic = nil
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
			continue
		}

		// 将defer标记为start
		d.started = true

		// 将defer的_panic标记为本次panic，如果后面出现新的panic（对应上面那个if函数 d.started），那么会找到本次panic，并且将其标记为中止
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		p.argp = unsafe.Pointer(getargp(0))
		
		// 调用defer函数
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		p.argp = nil

		// reflectcall did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		freedefer(d)
		
		if p.recovered {
			// 处理recovery
			atomic.Xadd(&runningPanicDefers, -1)

			gp._panic = p.link
			// 如果panic 被中止掉，那么从链表中将其删除
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { // must be done with signal
				gp.sig = 0
			}

			// 将堆栈信息传输给recovery
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)

	fatalpanic(gp._panic) // 不可恢复的panic，最后exit 返回2
	*(*int)(nil) = 0      // not reached
}
```


### fatalpanic

```go
//go:nosplit  跳过堆栈溢出检测
func fatalpanic(msgs *_panic) {
	pc := getcallerpc()
	sp := getcallersp()
	gp := getg()
	var docrash bool
	// Switch to the system stack to avoid any stack growth, which
	// may make things worse if the runtime is in a bad state.
	systemstack(func() {
		if startpanic_m() && msgs != nil {
			// There were panic messages and startpanic_m
			// says it's okay to try to print them.

			// startpanic_m set panicking, which will
			// block main from exiting, so now OK to
			// decrement runningPanicDefers.
			atomic.Xadd(&runningPanicDefers, -1)

			printpanics(msgs)  // 递归打印panic信息
		}

		docrash = dopanic_m(gp, pc, sp)
	})

	if docrash {
		// By crashing outside the above systemstack call, debuggers
		// will not be confused when generating a backtrace.
		// Function crash is marked nosplit to avoid stack growth.
		crash()
	}

	systemstack(func() {
		exit(2)
	})

	*(*int)(nil) = 0 // not reached
}
```
