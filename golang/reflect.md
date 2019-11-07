## 深入理解Reflect
Go是一门静态语言，我们可以使用反射对不同类型的对象进行操作。


### Type
我们可以通过TypeOf将一个对象转换成Type

```go
    t := reflect.TypeOf(1)
    fmt.Println(t) // int
```

首先我们看下Type的组成(下图中我们省略了一些字段)
```go
type Type interface {
    // 返回第i个方法，超过范围，则panic
	Method(int) Method

    // 通过名称获取方法
	MethodByName(string) (Method, bool)

    // 方法数量
	NumMethod() int

    // 类型名
	Name() string

    // 返回包路径（完全路径）
	PkgPath() string

    // 类似unsafe.Sizeof
	Size() uintptr

    // 字符串表示，可能包含短包名
	String() string

    // 类型
	Kind() Kind

    // 如果该类型实现了u的接口，则返回true
	Implements(u Type) bool

    // 返回元素类型，如果kind不是slice,channel,map,pointer, 则painc
	Elem() Type

    // 第几个字段
	Field(i int) StructField

	FieldByIndex(index []int) StructField

    // 根据name返回字段细腻系
	FieldByName(name string) (StructField, bool)

	FieldByNameFunc(match func(string) bool) (StructField, bool)

    // 返回第i个参数类型
	In(i int) Type

    // 返回array类型的长度，如果不是，则panic
	Len() int

    // 返回field的数量
	NumField() int

    // 返回参数的数量
	NumIn() int
}
```
可以看到Type是interface，接下来我们看下TypeOf
```go
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
func toType(t *rtype) Type {
	if t == nil {
		return nil
	}
	return t
}
```
golang里面接口类型都是通过emptyInterface来表示的，其中rtype表示变量的类型，word指向数据的地址
```go
type emptyInterface struct {
	typ  *rtype
	word unsafe.Pointer
}
```

### Value
我们可以通过ValueOf将一个类型转换成Value类型
```go
    t := reflect.ValueOf(1)
    fmt.Println(t) // 1
```
Value是一个结构体
```go
type Value struct {
	// value的type类型
	typ *rtype

    // 指针指向数据
	ptr unsafe.Pointer

	flag
}
```

#### ValueOf
```go
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

    // 从栈逃逸到堆上面
	escapes(i)

	return unpackEface(i)
}

func unpackEface(i interface{}) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// NOTE: don't read e.word until we know whether it is really a pointer or not.
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if ifaceIndir(t) {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```
可以看到ValueOf实际上也是先转成emptyInterface，然后再生成Value



### 用法
下面我们通过reflect的用法来具体分析下

#### 类型转换
下面的转换是int => interface{} => emptyInterface => Value => Interface => int
```go
    v := reflect.ValueOf(1)
	vv := v.Interface().(int)
```

#### 修改数据
```go
    x := 1
	vv := reflect.ValueOf(&x)
    p := vv.Elem()
	fmt.Println(p.CanSet())   // true
	p.SetInt(2)
	fmt.Println(x)   // 2
```

##### Elem
Elem返回v指针指向的实际变量，如果不是接口或者指针类型，会Panic
```go
func (v Value) Elem() Value {
	k := v.kind()
	switch k {
	case Interface:
		var eface interface{}
		if v.typ.NumMethod() == 0 {
			eface = *(*interface{})(v.ptr)
		} else {
			eface = (interface{})(*(*interface {
				M()
			})(v.ptr))
		}
		x := unpackEface(eface)
		if x.flag != 0 {
			x.flag |= v.flag.ro()
		}
		return x
	case Ptr:
		ptr := v.ptr
		if v.flag&flagIndir != 0 {
			ptr = *(*unsafe.Pointer)(ptr)
		}
		// The returned value's address is v's value.
		if ptr == nil {
			return Value{}
		}
		tt := (*ptrType)(unsafe.Pointer(v.typ))
		typ := tt.elem
		fl := v.flag&flagRO | flagIndir | flagAddr
		fl |= flag(typ.Kind())
		return Value{typ, ptr, fl}
	}
	panic(&ValueError{"reflect.Value.Elem", v.kind()})
}
```

#### Implements
我们可以通过Implements方法判断某些类型是否实现了全部的方法
```go
type Demo struct{}

func (d *Demo) Error() string {
	return "demo error"
}
func main() {

	r := reflect.TypeOf((*error)(nil)).Elem()  // 获取接口的类型
	dPtr := reflect.TypeOf(&Demo{})
	d := reflect.TypeOf(Demo{})
	fmt.Println(dPtr.Implements(r))   // true
	fmt.Println(d.Implements(r))      // false
}
```

##### 具体方法
```go
func (t *rtype) Implements(u Type) bool {
	// 如果是nil，则会panic
	if u == nil {
		panic("reflect: nil type passed to Type.Implements")
	}
	// 如果不是指针类型，会panic
	if u.Kind() != Interface {
		panic("reflect: non-interface type passed to Type.Implements")
	}
	return implements(u.(*rtype), t)
}
func implements(T, V *rtype) bool {
	if T.Kind() != Interface {
		return false
	}
	t := (*interfaceType)(unsafe.Pointer(T))
	// 如果u的方法数量为0，则返回true
	if len(t.methods) == 0 {
		return true
	}

	// 两种方式都一样，通过遍历V和T中的方法来判断是否满足条件，methods是有序的
	if V.Kind() == Interface {
		v := (*interfaceType)(unsafe.Pointer(V))
		i := 0
		for j := 0; j < len(v.methods); j++ {
			tm := &t.methods[i]
			tmName := t.nameOff(tm.name)
			vm := &v.methods[j]
			vmName := V.nameOff(vm.name)
			if vmName.name() == tmName.name() && V.typeOff(vm.typ) == t.typeOff(tm.typ) {
				if !tmName.isExported() {
					tmPkgPath := tmName.pkgPath()
					if tmPkgPath == "" {
						tmPkgPath = t.pkgPath.name()
					}
					vmPkgPath := vmName.pkgPath()
					if vmPkgPath == "" {
						vmPkgPath = v.pkgPath.name()
					}
					if tmPkgPath != vmPkgPath {
						continue
					}
				}
				if i++; i >= len(t.methods) {
					return true
				}
			}
		}
		return false
	}

	v := V.uncommon()
	if v == nil {
		return false
	}
	i := 0
	vmethods := v.methods()
	for j := 0; j < int(v.mcount); j++ {
		tm := &t.methods[i]
		tmName := t.nameOff(tm.name)
		vm := vmethods[j]
		vmName := V.nameOff(vm.name)
		if vmName.name() == tmName.name() && V.typeOff(vm.mtyp) == t.typeOff(tm.typ) {
			if !tmName.isExported() {
				tmPkgPath := tmName.pkgPath()
				if tmPkgPath == "" {
					tmPkgPath = t.pkgPath.name()
				}
				vmPkgPath := vmName.pkgPath()
				if vmPkgPath == "" {
					vmPkgPath = V.nameOff(v.pkgPath).name()
				}
				if tmPkgPath != vmPkgPath {
					continue
				}
			}
			if i++; i >= len(t.methods) {
				return true
			}
		}
	}
	return false
}
```

#### 方法调用
我们如何通过反射去调用方法
```go
func Sum(a, b int) int {
	return a + b
}
func main() {
	v := reflect.ValueOf(Sum)
	fmt.Println(v.Kind()) // func

	fmt.Println(v.Type().NumIn()) // 参数数量2
	result := v.Call([]reflect.Value{reflect.ValueOf(1), reflect.ValueOf(2)})

	fmt.Println(result[0].Int()) // 3
}
```

##### Call
```go
func (v Value) Call(in []Value) []Value {
	v.mustBe(Func)   //  必须是盘数
	v.mustBeExported()  // 必须是可导出的
	return v.call("Call", in)
}
func (v Value) call(op string, in []Value) []Value {
	// 获取type的值
	t := (*funcType)(unsafe.Pointer(v.typ))
	var (
		fn       unsafe.Pointer
		rcvr     Value
		rcvrtype *rtype
	)
	if v.flag&flagMethod != 0 {
		rcvr = v
		rcvrtype, t, fn = methodReceiver(op, v, int(v.flag)>>flagMethodShift)
	} else if v.flag&flagIndir != 0 {
		fn = *(*unsafe.Pointer)(v.ptr)
	} else {
		fn = v.ptr
	}

	// 检查是否nil
	if fn == nil {
		panic("reflect.Value.Call: call of nil function")
	}
	// 判断是否是CallSlice，比如len(arr), in是arr[0], arr[1], arr[2]...
	isSlice := op == "CallSlice"
	n := t.NumIn()
	if isSlice {
		if !t.IsVariadic() {
			panic("reflect: CallSlice of non-variadic function")
		}
		if len(in) < n {
			panic("reflect: CallSlice with too few input arguments")
		}
		if len(in) > n {
			panic("reflect: CallSlice with too many input arguments")
		}
	} else {
		if t.IsVariadic() {
			n--
		}
		if len(in) < n {
			panic("reflect: Call with too few input arguments")
		}
		if !t.IsVariadic() && len(in) > n {
			panic("reflect: Call with too many input arguments")
		}
	}
	for _, x := range in {
		if x.Kind() == Invalid {
			panic("reflect: " + op + " using zero Value argument")
		}
	}
	for i := 0; i < n; i++ {
		if xt, targ := in[i].Type(), t.In(i); !xt.AssignableTo(targ) {
			panic("reflect: " + op + " using " + xt.String() + " as type " + targ.String())
		}
	}
	if !isSlice && t.IsVariadic() {
		// prepare slice for remaining values
		m := len(in) - n
		slice := MakeSlice(t.In(n), m, m)
		elem := t.In(n).Elem()
		for i := 0; i < m; i++ {
			x := in[n+i]
			if xt := x.Type(); !xt.AssignableTo(elem) {
				panic("reflect: cannot use " + xt.String() + " as type " + elem.String() + " in " + op)
			}
			slice.Index(i).Set(x)
		}
		origIn := in
		in = make([]Value, n+1)
		copy(in[:n], origIn)
		in[n] = slice
	}

	nin := len(in)
	// 检查参数
	if nin != t.NumIn() {
		panic("reflect.Value.Call: wrong argument count")
	}
	// 开始进入准备阶段
	nout := t.NumOut()

	// 计算参数和返回值锁需要占的空间大小
	frametype, _, retOffset, _, framePool := funcLayout(t, rcvrtype)

	// Allocate a chunk of memory for frame.
	var args unsafe.Pointer
	if nout == 0 {
		args = framePool.Get().(unsafe.Pointer)
	} else {
		// 如果有返回参数就不能使用pool，需要单独分配args
		args = unsafe_New(frametype)
	}
	off := uintptr(0)

	// 拷贝输入到args
	if rcvrtype != nil {
		storeRcvr(rcvr, args)
		off = ptrSize
	}
	for i, v := range in {
		v.mustBeExported()
		targ := t.In(i).(*rtype)
		a := uintptr(targ.align)
		off = (off + a - 1) &^ (a - 1)
		n := targ.size
		if n == 0 {
			// Not safe to compute args+off pointing at 0 bytes,
			// because that might point beyond the end of the frame,
			// but we still need to call assignTo to check assignability.
			v.assignTo("reflect.Value.Call", targ, nil)
			continue
		}
		addr := add(args, off, "n > 0")
		v = v.assignTo("reflect.Value.Call", targ, addr)
		if v.flag&flagIndir != 0 {
			typedmemmove(targ, addr, v.ptr)
		} else {
			*(*unsafe.Pointer)(addr) = v.ptr
		}
		off += n
	}

	// 开始调用
	call(frametype, fn, args, uint32(frametype.size), uint32(retOffset))

	// For testing; see TestCallMethodJump.
	if callGC {
		runtime.GC()
	}

	var ret []Value
	if nout == 0 {
		// 清除args，将args放回到pool中
		typedmemclr(frametype, args)
		framePool.Put(args)
	} else {
		// Zero the now unused input area of args,
		// because the Values returned by this function contain pointers to the args object,
		// and will thus keep the args object alive indefinitely.
		typedmemclrpartial(frametype, args, 0, retOffset)

		// 将返回值写入到ret中
		ret = make([]Value, nout)
		off = retOffset
		for i := 0; i < nout; i++ {
			tv := t.Out(i)
			a := uintptr(tv.Align())
			off = (off + a - 1) &^ (a - 1)
			if tv.Size() != 0 {
				fl := flagIndir | flag(tv.Kind())
				ret[i] = Value{tv.common(), add(args, off, "tv.Size() != 0"), fl}
				// Note: this does introduce false sharing between results -
				// if any result is live, they are all live.
				// (And the space for the args is live as well, but as we've
				// cleared that space it isn't as big a deal.)
			} else {
				// For zero-sized return value, args+off may point to the next object.
				// In this case, return the zero value instead.
				ret[i] = Zero(tv)
			}
			off += tv.Size()
		}
	}

	return ret
}
```


#### 访问结构体私有属性
引入其他包中结构体是无法直接访问其私有属性的，我们可以通过reflect进行访问
```go
// 这个结构体在demo/demo.go中
type Demo struct {
    name    string
    age    int8
}


// main.go

package main

import (
	"demo"
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	var d demo.Demo
	v := reflect.ValueOf(&d)
	elem := v.Elem()

	fmt.Println(elem.Field(0), elem.Field(1)) // "", 0
}
```

##### 如果我们想修改私有变量的值怎么变
```go
func change(d *demo.Demo) {
	dd := unsafe.Pointer(d)
	v := uintptr(0) // 第一个字段，offset为0
	p := (*string)(unsafe.Pointer(uintptr(dd) + v))  
	*p = "a"   // 这时name就变成a了

	// uintptr(16) 是因为name是string类型，需要占两个字节
	age := (*int)(unsafe.Pointer(uintptr(dd) + uintptr(16)))

	*age = 11 // age就变成 11了
}
```

