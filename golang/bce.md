## Go 边界检查消除
在Go和其他语言中，都会对数组用索引获取值`a[i]`和`a[i:j]`进行检查，以免越界。今天我们介绍的就是边界检查消除（Bounds Check Elimination），在Go1.7之后的版本，采用新的编译后端，基于SSA（static single-assignment form， 静态单复制形式），SSA使用BCE（边界检查消除）和CSE（公共子表达式消除）来让GO编译器有效的优化代码

### 边界检查消除
接下来我们看个例子了解下边界检查消除到底会影响些什么？
```go
func BenchmarkSumForward(b *testing.B) {
	nums := []int{}
	for i := 0; i < 5; i++ {
		nums = append(nums, i)
	}

	for n := 0; n < b.N; n++ {
		sum := nums[0] + nums[1] + nums[2] + nums[3] + nums[4]
		_ = sum
	}
}

func BenchmarkSumBackward(b *testing.B) {
	nums := []int{}
	for i := 0; i < 5; i++ {
		nums = append(nums, i)
	}

	for n := 0; n < b.N; n++ {
		sum := nums[4] + nums[3] + nums[2] + nums[1] + nums[0]
		_ = sum
	}
}
```
我们运行下benckmark
```bash
BenchmarkSumForward-4   	2000000000	         0.85 ns/op	       0 B/op	       0 allocs/op
BenchmarkSumBackward-4   	2000000000	         0.29 ns/op	       0 B/op	       0 allocs/op
```
看似差不多的两段代码，为什么后者比前者要高效很多，我们对比下两个Benchmark不同之处，后面那个函数sum赋值和前面的赋值相反，为什么顺序写反，就有这么大的性能差异，这个就是我们今天聊的边界检查消除，编译器把`BenchmarkSumBackward`的边界检查消除取消了，所以性能才有了提升，那么我们应该如何利用边界检查消除，提升我们的程序性能呢？接下来我们聊一下边界检查消除的情况

### 边界检查消除

#### 重复的检查

```go{class=line-numbers}
package main

func main() {
	var a []int

	for i := 0; i < 5; i++ {
		a = append(a, i)
	}

	_ = a[1]
	_ = a[1]  // 不会边界检查

    _ = a[:2]
    _ = a[:2] // 不会边界检查

    _ = a[2:]
    _ = a[2:] // 不会边界检查

}

```
我们使用`go build -gcflags="-d=ssa/check_bce/debug=1"`可以查看是否引入了边界检查，可以看到只有第10行的进行了边界检查，在1.7之前的版本第11行也会做边界检查

```bash
# command-line-arguments
./bce_demo.go:10:7: Found IsInBounds
```

#### 固定长度slice
```go{class=line-numbers}
package main

func f1(i int) {
	var a [17]int
	_ = a[i&5] // 0 <= i&5 <=5 <17, 边界检查清除
	_ = a[i%5] // i有可能是负数，还要边界检查
}
func main() {}
```

#### 常量索引

```go
package main

func f1() {
	var a []int
	if 1 < len(a) { 
		_ = a[1]  // 0 < 1 and 1 < len(a)，边界检查清除
	}
}
func main() {}
```

#### 常量索引和固定长度数组

```go
package main

func f1() {
	var a [10]int
	_ = a[5] // 0 <= 5 < 10 == len(a), 边界检查清除
}
func main() {}
```

#### 其他约束检查
`a[i:j]`会产生两个边界检查: `for 0 <= i <=j and for 0 <= j <= cap(a)`. 有些情况下我们可以移除这些边界检查

```go
	var a []int
	_ = a[i:len(a)] // 第二个边界检查被移除 0 <= len(a) <= cap(a)

	...
	var a []int
	_ = a[:len(a)>>1]  // 第一个边界检查被移除， 0 <= 0 <= len(a) >> 1 因为 len(a) >> 1 >=0

	...
	var a []int
	_ = a[:len(b)]    // 第一个边界检查被移除，0 <= 0 <= len(a) 
```

#### 基于变量的BCE检查
有些边界检查会在index迭代slice和string后清除

```go
	var a []int
	for i:= range a{
		_ = a[i]   // 移除, i是一个迭代后的索引
		_ = a[i:]  // 移除
		_ = a[:i]  // 移除
	}
```

```go
	var a []int
	for i:=0; i < len(a); i++{
		_ = a[i]   // 移除, i是一个迭代后的索引
		_ = a[i:]  // 移除
		_ = a[:i]  // 移除
	}
```

```go
	var a []int
	for i := range a{
		_ = a[:i+1]   // 移除, i是一个迭代后的索引
		_ = a[i+1:]   // 移除
	}
```

#### 递减的常量索引

```go
	var a []int
	_ = a[3]   // 一次边界检查
	_, _, _, _ = a[0], a[1], a[2], a[3]  // 没有边界检查

	//...
	a = a[:3:len(a)] // 一次边界检查
	_, _, _, _ = a[0], a[1], a[2], a[3] // 没有边界检查
	
	//...
	if len(a) < 3{
		_, _, _ = a[0], a[1], a[3]  // 没有边界检查
	}
```

### 总结
没想到一个小小的边界值也能有这么多处理，真是让人惊叹Go语言的魅力！