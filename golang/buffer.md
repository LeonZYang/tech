## 深入理解Buffer
Buffer是个可以供读写的可变长字节缓冲区

### 数据结构

```go
type Buffer struct {
    // buf[off : len(buf)] 内容
    buf      []byte // contents are the bytes buf[off : len(buf)]
    // 读取在&buf[off], 写入在&buf[len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // 最后一次读取操作
}

const (
    // 读取操作
    opRead      readOp = -1 // Any other read operation.
    // 非读操作
    opInvalid   readOp = 0  // Non-read operation.
    // 读取第一个字节
	opReadRune1 readOp = 1  // Read rune of size 1.
	opReadRune2 readOp = 2  // Read rune of size 2.
	opReadRune3 readOp = 3  // Read rune of size 3.
	opReadRune4 readOp = 4  // Read rune of size 4.
)
```

### 操作方法

#### Bytes
返回Buffer中未读取的字节
```go
func (b *Buffer) Bytes() []byte { return b.buf[b.off:] }
```

#### Len
返回未读取的字节长度，b.Len() == len(b.Bytes())
```go
func (b *Buffer) Len() int { return len(b.buf) - b.off }
```

#### Cap
Cap返回Buffer当前的容量长度
```go
func (b *Buffer) Cap() int { return cap(b.buf) }
```

#### Truncate
Truncate 会丢弃除了未读的数据， 如果n==0 会丢弃所有数据
```go
func (b *Buffer) Truncate(n int) {
	if n == 0 {
		b.Reset()
		return
	}
	b.lastRead = opInvalid
	if n < 0 || n > b.Len() {
		panic("bytes.Buffer: truncation out of range")
	}
	b.buf = b.buf[:b.off+n]
}
```

#### Reset
Reset清空缓存区数据
```go
func (b *Buffer) Reset() {
	b.buf = b.buf[:0]
	b.off = 0
	b.lastRead = opInvalid
}
```

#### Grow
Grow 会增加buffer的容量，n不得为负数
```go
func (b *Buffer) Grow(n int) {
	if n < 0 {
		panic("bytes.Buffer.Grow: negative count")
	}
	m := b.grow(n)
	b.buf = b.buf[:m]
}
```

#### grow
grow增加buffer的容量，保证有n个字节可以写入
```go
func (b *Buffer) grow(n int) int {
	m := b.Len()
    // 如果buffer为空，执行reset获取空间
	if m == 0 && b.off != 0 {
		b.Reset()
	}

    // 尝试快速扩容
	if i, ok := b.tryGrowByReslice(n); ok {
		return i
    }
    // smallBufferSize = 64
    // buf为空，并且扩容的大小过小
	if b.buf == nil && n <= smallBufferSize {
		b.buf = make([]byte, n, smallBufferSize)
		return 0
	}
    c := cap(b.buf)
    // 如果buf长度+n小于等于buf容量的一半，这种只需向下滑动即可
	if n <= c/2-m {
		// We can slide things down instead of allocating a new
		// slice. We only need m+n <= c to slide, but
		// we instead let capacity get twice as large so we
        // don't spend all our time copying.
        // 丢弃已读的数据
		copy(b.buf, b.buf[b.off:])
	} else if c > maxInt-c-n {
        // 要分配的数据太大了
		panic(ErrTooLarge)
	} else {
        // 没有足够的空间，直接分配，分配后的大小是2*c + n
        buf := makeSlice(2*c + n)
        // 丢弃已读数据
		copy(buf, b.buf[b.off:])
		b.buf = buf
	}

    // 重置off
	b.off = 0
	b.buf = b.buf[:m+n]
	return m
}
```

#### tryGrowByReslice
tryGrowByReslice返回可写入字节的索引以及是否扩容成功，快速扩容方案
```go
func (b *Buffer) tryGrowByReslice(n int) (int, bool) {
    // 如果buf空闲的数量大于要扩容的大小
	if l := len(b.buf); n <= cap(b.buf)-l {
        // 快速扩容，reslice
		b.buf = b.buf[:l+n]
		return l, true
	}
	return 0, false
}
```

#### Write
Write写入数据到buffer中，必要情况下会扩容buffer，如果err == nil，n == len(p)
```go
func (b *Buffer) Write(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	m, ok := b.tryGrowByReslice(len(p))
	if !ok {
		m = b.grow(len(p))
	}
	return copy(b.buf[m:], p), nil
}
```

#### ReadFrom
ReadFrom从r中读取直到EOF，并append到buffer中，如果需要，会扩容buffer。返回的n是读取的字节数，除了EOF的错误都会fan'hui
```go
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
	b.lastRead = opInvalid
	for {
		i := b.grow(MinRead) // MinRead = 512
		b.buf = b.buf[:i]
		m, e := r.Read(b.buf[i:cap(b.buf)])
		if m < 0 {
			panic(errNegativeRead)
		}

		b.buf = b.buf[:i+m]
		n += int64(m)
		if e == io.EOF {
			return n, nil // e is EOF, so return nil explicitly
		}
		if e != nil {
			return n, e
		}
	}
}

```

#### WriteTo
WriteTo 写入数据到w中，直接buffer中数据耗尽，返回写入的字节数和写入过程中遇到的错误
```go
func (b *Buffer) WriteTo(w io.Writer) (n int64, err error) {
	b.lastRead = opInvalid
	if nBytes := b.Len(); nBytes > 0 {
		m, e := w.Write(b.buf[b.off:])
		if m > nBytes {
			panic("bytes.Buffer.WriteTo: invalid Write count")
		}
		b.off += m
		n = int64(m)
		if e != nil {
			return n, e
		}
		// all bytes should have been written, by definition of
		// Write method in io.Writer
		if m != nBytes {
			return n, io.ErrShortWrite
		}
	}
	// Buffer 已经写完，reset
	b.Reset()
	return n, nil
}

```

#### Read
Read会读取下n个字节直接buffer耗尽，返回读取的字节和读取遇到的错误，如果buffer为空，错误返回EOF
```go
func (b *Buffer) Read(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	if b.empty() {
		// Buffer is empty, reset to recover space.
		b.Reset()
		if len(p) == 0 {
			return 0, nil
		}
		return 0, io.EOF
	}
	n = copy(p, b.buf[b.off:])
	b.off += n
	if n > 0 {
		b.lastRead = opRead
	}
	return n, nil
}
```

#### Next
Next返回下n个字节数据，如果buffer未读字节数小于n，则返回整个buffer，
```go
func (b *Buffer) Next(n int) []byte {
	b.lastRead = opInvalid
	m := b.Len()
	if n > m {
		n = m
	}
	data := b.buf[b.off : b.off+n]
	b.off += n
	if n > 0 {
		b.lastRead = opRead
	}
	return data
}
```

#### ReadRune
```go
func (b *Buffer) ReadRune() (r rune, size int, err error) {
	if b.empty() {
		// Buffer is empty, reset to recover space.
		b.Reset()
		return 0, 0, io.EOF
	}
	c := b.buf[b.off]
	if c < utf8.RuneSelf {
		b.off++
		b.lastRead = opReadRune1
		return rune(c), 1, nil
	}
	r, n := utf8.DecodeRune(b.buf[b.off:])
	b.off += n
	b.lastRead = readOp(n)
	return r, n, nil
}
```

### 总结
buffer提供了可扩容的缓冲池，如果数据比较大，我们已经提前申请好相应的buffer，以免buffer多次扩容。