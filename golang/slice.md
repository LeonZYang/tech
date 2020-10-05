## 深入理解Slice

Slice 就是我们常说的切片，Slice在内存中占用2个字节


###用法
```go
func main(){
    var (
        a []int
        b [3]int // 数组，是单独一个类型
        c = make([]int, 3)
    )

    a = append(a, 1)
    b[0] = 1   // 元素直接填充0， 固定长度，append 会报错
    c[0] = 1   // 元素填充0， 可以append，但是会在直接追加到队尾，可以使用make([]int, 0, 3)
    fmt.Println(a, b, c)
}
```

### 结构

```go
type slice struct {
	array unsafe.Pointer // 指向所引用的数组
	len   int         // 当前切片的长度
	cap   int         // 当前切片的容量， cap >= len, 否则会报panic:"makeslice: cap out of range"
}
```

### 理解Len和Cap
下面以几个例子说明下
```go
    var (
        a []int
        b [3]int 
        c = make([]int, 3)
    )

    fmt.Println(len(a), cap(a))   // 0, 0
    fmt.Println(len(b), cap(b))   // 3, 3
    fmt.Println(len(c), cap(c))   // 3, 3

    // 存储数据 
    a = append(a, 1)
    b[0] = 1
    c[0] = 1

    fmt.Println(len(a), cap(a))   // 1, 1
    fmt.Println(len(b), cap(b))   // 3, 3
    fmt.Println(len(c), cap(c))   // 3, 3

    // 向c中append数据
    c = append(c, 2)
    fmt.Println(len(c), cap(c))   // 4, 6


    // 思考？
    // var d = make([]int, 0, 3) len = ? and cap = ? 
```

从上面的例子，可以看出，b长度和容量是不可变的， c的长度和容量是相等的

### slice 扩容规则
slice扩容有下面几个规则
 - 如果新的容量大于旧的Slice的长度2倍，则扩容为当前Slice的2倍
 - 如果旧Slice的长度低于1024，则以两倍扩容，如果大于1024，则按旧Slice长度的1/4进行扩容
```go
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            // Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {
                newcap = cap
            }
        }
    }
```

### 切片陷阱
```go
    var a = []int{1, 2, 3}
    fmt.Printf("%v\n", *(*unsafe.Pointer)(unsafe.Pointer(&a))) // 0xc00001c090
    b := a[1:2]
    fmt.Printf("%v\n", *(*unsafe.Pointer)(unsafe.Pointer(&b))) // 0xc00001c098
    b[0] = 11  // 此时由于还是引用同一地址，不仅b[0]是2， a[1] 同样也变成11
```
可以看到切片后的地址相差了8个字节，也就是一个int，还是引用相同的一块内存地址

```go
    var a = []int{1, 2, 3}

    b := a[1:2]
    b = append(b, 4)
    b = append(b, 5)
    b = append(b, 6) // 触发扩容，生成新的Slice
    b[0] = 11  // 此时就只有b[0]变成了11， a[0]还是1
```


### Slice copy
从源Slice将数据拷贝到目的Slice中，拷贝的Slice大小是dst的大小
```go
func Copy(dst, src []T) int
```

```go
func slicecopy(to, fm slice, width uintptr) int {
	if fm.len == 0 || to.len == 0 {  // 源和目的长度都为0 直接返回
		return 0
	}

	n := fm.len
	if to.len < n {    // 如果目的slice大小小于源silce的大小，按目的的大小算
		n = to.len
	}

	if width == 0 {
		return n
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicecopy)
		racewriterangepc(to.array, uintptr(n*int(width)), callerpc, pc)
		racereadrangepc(fm.array, uintptr(n*int(width)), callerpc, pc)
	}
	if msanenabled {
		msanwrite(to.array, uintptr(n*int(width)))
		msanread(fm.array, uintptr(n*int(width)))
	}

	size := uintptr(n) * width
    if size == 1 { // common case worth about 2x to do here
        // TODO: is this still worth it with new memmove impl?
        // 只有1个元素，直接拷贝指针
		*(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
	} else {
        // 从fm拷贝size个字节到to
		memmove(to.array, fm.array, size)
	}
	return n
}

```


### 小陷阱
```go
    var a = make([]int, 3)
    var b = make([]int, 0, 3)
    a = append(a, 1)
    b = append(b, 1)

    fmt.Println(a) // [0, 0, 0, 1]
    fmt.Println(b) // [1]  
```
make 第二个参数是len，第三个参数是cap，如果不传第三个参数，那么cap = len，所以就会有下面的输出


### 其他
slice切片的时候，可以设定切片后的cap

```go
    s := make([]uint16, 5)
    s[0] = 1
    s[1] = 2
    s[2] = 3
    s[3] = 4
    s[4] = 5

    // 5 就是新切片的cap，不能大于原始切片的cap，否则会报panic: runtime error: slice bounds out of range
    v:= s[0:2:5]
```


