## 深入理解context
context是一个服务上下文切换的包，我们可以定义deadline或者手动cancel对一些派生的goroutine进行控制，比较常见的就是http服务，一个http可能会启动N个goroutine，那么当http request取消后，如果快速关闭这些goroutine，还有种场景就是进行上下文变量传递。

### 数据结构

#### Context
Context是个接口类型
```go
type Context interface {
    // 返回当前任务取消的时间， ok表示是否设置deadline
	Deadline() (deadline time.Time, ok bool)

    // done返回一个只读channel，当cancel被调用或者deadline过期后，会close 这个channnel
	Done() <-chan struct{}

    // 如果Done没有被关闭，返回nil，否则返回一个非nil的error
	Err() error

    // value返回这个上下文关联key的value，如果没有设置，则返回nil
	Value(key interface{}) interface{}
}
```

#### Backgroud和TODO
emptyCtx是一个无法cancel的Context，backgroud和todo是两个emptyCtx的指针，如果你不确定你的Context有什么用，那么就用context.TODO()
```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```


#### cancelCtx
cancelCtx实现了取消，如果取消，那么children(实现了cancel)的也会被取消
```go
type cancelCtx struct {
	Context

	// 锁
	mu       sync.Mutex            // protects following fields
	// 首次取消会close
	done     chan struct{}         // created lazily, closed by first cancel call
	// 首次cancel，会被设置成nil
	children map[canceler]struct{} // set to nil by the first cancel call
	// 首次取消是个非nil的error·
	err      error                 // set to non-nil by the first cancel call
}

```

#### timerCtx
timerCtx内嵌了cancelCtx
```go
type timerCtx struct {
	cancelCtx
	// 被cancelCtx lock保护
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

```

#### valueCtx
valueCtx有一对key和val
```go
type valueCtx struct {
	Context
	key, val interface{}
}
```

### 方法


#### WithCancel
WithCancel创建一个cancelCtx的context
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

#### WithDeadline
WithDeadline 返回一个deadline的context，当parent done被close，deadline过期和cancel，这个context的Done会被closed
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        // parent的deadline早于当前的deadline，肯定parent先达到expired
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}

```

#### WithTimeout
WithTimeout相当于WithDeadline(parent, time.Now().Add(timeout))
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

#### WithValue
WithValue返回一个关联key的context, key必须是个可比较得指针或者对象，
```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflect.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

### 其他

#### propagateCancel
propagateCancel用来检查parent状态和启动监听任务
```go
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // parent is never canceled
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent 已经关闭了
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			// 启动监听parent和child的goroutine
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

#### parentCancelCtx
从下至上直到获取cancelCtx
```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context
		default:
			return nil, false
		}
	}
}
```

总结：总体上看context源码还是比较简单的，在实际应用中context也是非常有用的