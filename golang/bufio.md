## 深入理解bufio
io本身操作由于直接对磁盘操作，所以效率比较低，bufio提供了缓存区来缓解这个问题。接下来我们看下bufio。

### Reader

#### 数据结构
Reader相当于有缓存的io.Reader
```go
type Reader struct {
	buf          []byte
	rd           io.Reader // 体用读取的Reader的
	r, w         int       // buf读取和写入的位置
	err          error
	lastByte     int // 最后读取的字节，-1为不可用
	lastRuneSize int // 最后读取UnreadRune rune的size
}
```

#### NewReaderSize
NewReaderSize新建一个Reader
```go
func NewReaderSize(rd io.Reader, size int) *Reader {
    // 如果已经是个Reader并且当前的缓存大小大于size
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
    }
    // minReadBufferSize = 16
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}
```

#### NewReader
NewReader会创建个4096大小的buf
```go
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}
```

#### Read
```go
func (b *Reader) Read(p []byte) (n int, err error) {
	n = len(p)
	if n == 0 {
		return 0, b.readErr()
	}
	if b.r == b.w {
		if b.err != nil {
			return 0, b.readErr()
        }
        
		if len(p) >= len(b.buf) {
            // 如果要读取的数据大于缓存，直接读取
			n, b.err = b.rd.Read(p)
			if n < 0 {
				panic(errNegativeRead)
			}
			if n > 0 {
                // 更新最后读取的字节
				b.lastByte = int(p[n-1])
				b.lastRuneSize = -1
			}
			return n, b.readErr()
		}

        // 当len(p) < len(b.buf)，读取到buf
		b.r = 0
		b.w = 0
		n, b.err = b.rd.Read(b.buf)
		if n < 0 {
			panic(errNegativeRead)
		}
		if n == 0 {
			return 0, b.readErr()
        }
        // 更新写入的位置
		b.w += n
	}

    // 从缓存区拷贝
	n = copy(p, b.buf[b.r:b.w])
    b.r += n
    // 更新读取最后的字节
	b.lastByte = int(b.buf[b.r-1])
	b.lastRuneSize = -1
	return n, nil
}
```

#### Reset
Reset清空数据
```go
func (b *Reader) Reset(r io.Reader) {
	b.reset(b.buf, r)
}

func (b *Reader) reset(buf []byte, r io.Reader) {
	*b = Reader{
		buf:          buf,
		rd:           r,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}
```


### Writer
Writer对io.Write的封装
#### 数据结构
```go
type Writer struct {
	err error
	buf []byte
	n   int  // 当前缓存的位置
	wr  io.Writer
}
```

#### NewWriterSize
新建Writer，写入数据后，应该调用Flush将数据写入io.Writer
```go
func NewWriterSize(w io.Writer, size int) *Writer {
	// Is it already a Writer?
	b, ok := w.(*Writer)
	if ok && len(b.buf) >= size {
		return b
    }
    // defaultBufSize = 4096
	if size <= 0 {
		size = defaultBufSize
	}
	return &Writer{
		buf: make([]byte, size),
		wr:  w,
	}
}

// NewWriter returns a new Writer whose buffer has the default size.
func NewWriter(w io.Writer) *Writer {
	return NewWriterSize(w, defaultBufSize)
}
```

#### Write
Write写入数据
```go
func (b *Writer) Write(p []byte) (nn int, err error) {
    // 写入的数量大于当前可用的空间
	for len(p) > b.Available() && b.err == nil {
		var n int
        // Buffered 返回写入缓存区的字节数
		if b.Buffered() == 0 {
            // 大量写入，清空buf，将数据直接从p写入
			n, b.err = b.wr.Write(p)
		} else {
            // 如果buf还有剩余，将数据一部分写入到buf中
			n = copy(b.buf[b.n:], p)
            b.n += n
            // 写入数据后再将数据刷新
			b.Flush()
		}
        nn += n
        // 写入剩下的
		p = p[n:]
	}
	if b.err != nil {
		return nn, b.err
    }
    // 有空闲的，直接写入
    n := copy(b.buf[b.n:], p)
    // 更新当前写入的位置
	b.n += n
	nn += n
	return nn, nil
}
```

#### Available
Available返回还有多少字节没有使用
```go
func (b *Writer) Available() int { return len(b.buf) - b.n }
```

#### Flush
Flush刷新缓存数据到io.Writer
```go
func (b *Writer) Flush() error {
	if b.err != nil {
		return b.err
	}
	if b.n == 0 {
		return nil
    }
    // 将当前buf的数据写入
	n, err := b.wr.Write(b.buf[0:b.n])
	if n < b.n && err == nil {
		err = io.ErrShortWrite
	}
	if err != nil {
        // 写入失败
		if n > 0 && n < b.n {
            // 有数据没有写入，将没有写入的继续更新到buf中
			copy(b.buf[0:b.n-n], b.buf[n:b.n])
        }
		b.n -= n
		b.err = err
		return err
	}
	b.n = 0
	return nil
}
```

#### Reset
Reset 清除数据
```go
func (b *Writer) Reset(w io.Writer) {
	b.err = nil
	b.n = 0
	b.wr = w
}
```