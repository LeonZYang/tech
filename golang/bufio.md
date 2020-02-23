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

### Scanner
Scanner读取数据，并根据split函数对数据拆分，经常用来处理数据流

#### 数据结构
```go
type Scanner struct {
	// Reader
	r            io.Reader // The reader provided by the client.
	// split 函数
	split        SplitFunc // The function to split the tokens.
	// MaxScanTokenSize = 64 * 1024
	maxTokenSize int       // Maximum size of a token; modified by tests.
	// 最后一个token
	token        []byte    // Last token returned by split.
	// buf池
	buf          []byte    // Buffer used as argument to split.
	// 开始
	start        int       // First non-processed byte in buf.
	// 结束
	end          int       // End of data in buf.
	// 错误
	err          error     // Sticky error.
	empties      int       // Count of successive empty tokens.
	scanCalled   bool      // Scan has been called; buffer is in use.
	// 扫描结束
	done         bool      // Scan has finished.
}
```

##### SplitFunc
SplitFunc将输入源进行分割返回，输入的参数atEOF代表是否有错误，返回的参数advance是前进的字节数，token是截取的token
```go
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

#### NewScanner
NewScanner 返回一个Scanner
```go
// NewScanner returns a new Scanner to read from r.
// The split function defaults to ScanLines.
func NewScanner(r io.Reader) *Scanner {
	return &Scanner{
		r:            r,
		split:        ScanLines,
		maxTokenSize: MaxScanTokenSize,
	}
}
```

#### ScanLines
ScanLines是按照换行符进行分割
```go
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error) {
	// 如果错误，并且data为空，则返回
	if atEOF && len(data) == 0 {
		return 0, nil, nil
	}
	if i := bytes.IndexByte(data, '\n'); i >= 0 {
		// 返回一个完成行的数据， dropCR 用来删除最后一个是\r的
		return i + 1, dropCR(data[0:i]), nil
	}
	// 如果遇到一个EOF，返回一个
	if atEOF {
		return len(data), dropCR(data), nil
	}
	// Request more data.
	return 0, nil, nil
}
```

#### Scan
Scan扫描下一个token，可以通过Scaner的Text和Bytes方法，当读取到末尾或者是遇到错误，返回false。返回false后，Err方法会返回发生的错误，除了EOF。
```go
func (s *Scanner) Scan() bool {
	// 扫描结束
	if s.done {
		return false
	}
	// 扫描开始
	s.scanCalled = true
	// 循环直到有个Token
	for {
		// See if we can get a token with what we already have.
		// If we've run out of data but have an error, give the split function
		// a chance to recover any remaining, possibly empty token.
		// 获取一个Token，如果这里有个错误，
		if s.end > s.start || s.err != nil {
			// 分割
			advance, token, err := s.split(s.buf[s.start:s.end], s.err != nil)
			if err != nil {
				// ErrFinalToken 是个特殊的错误，代表Scan可以停止
				if err == ErrFinalToken {
					s.token = token
					s.done = true
					return true
				}
				s.setErr(err)
				return false
			}
			// start+= advance，如果失败则返回false
			if !s.advance(advance) {
				return false
			}
			// 扫描的Token
			s.token = token
			if token != nil {
				if s.err == nil || advance > 0 {
					s.empties = 0
				} else {
					// Returning tokens not advancing input at EOF.
					s.empties++
					if s.empties > maxConsecutiveEmptyReads {
						panic("bufio.Scan: too many empty tokens without progressing")
					}
				}
				return true
			}
		}
		// We cannot generate a token with what we are holding.
		// If we've already hit EOF or an I/O error, we are done.
		// 产生token失败，如果已经遇到EOF或者I/O错误，返回
		if s.err != nil {
			// Shut it down.
			s.start = 0
			s.end = 0
			return false
		}
		// Must read more data.
		// First, shift data to beginning of buffer if there's lots of empty space
		// or space is needed.
		// 当前start已经超过buf的一半或者end等于buf，需要将start:end数据段拷贝到0:end-start段
		if s.start > 0 && (s.end == len(s.buf) || s.start > len(s.buf)/2) {
			copy(s.buf, s.buf[s.start:s.end])
			s.end -= s.start
			s.start = 0
		}

		// 如果buffer已经满了，扩容
		if s.end == len(s.buf) {
			// Guarantee no overflow in the multiplication below.
			const maxInt = int(^uint(0) >> 1)
			// 如果当前buf超过maxTokenSize或者溢出
			if len(s.buf) >= s.maxTokenSize || len(s.buf) > maxInt/2 {
				s.setErr(ErrTooLong)
				return false
			}
			// 扩容2倍
			newSize := len(s.buf) * 2
			if newSize == 0 {
				newSize = startBufSize
			}
			if newSize > s.maxTokenSize {
				newSize = s.maxTokenSize
			}
			newBuf := make([]byte, newSize)
			copy(newBuf, s.buf[s.start:s.end])
			s.buf = newBuf
			s.end -= s.start
			s.start = 0
		}
		// Finally we can read some input. Make sure we don't get stuck with
		// a misbehaving Reader. Officially we don't need to do this, but let's
		// be extra careful: Scanner is for safe, simple jobs.

		for loop := 0; ; {
			n, err := s.r.Read(s.buf[s.end:len(s.buf)])
			s.end += n
			if err != nil {
				s.setErr(err)
				break
			}
			if n > 0 {
				s.empties = 0
				break
			}
			loop++
			// maxConsecutiveEmptyReads=100
			if loop > maxConsecutiveEmptyReads {
				s.setErr(io.ErrNoProgress)
				break
			}
		}
	}
}
```

#### Buffer
调整缓存大小，主要要再Scan开始前调用，否则会panic
```go
func (s *Scanner) Buffer(buf []byte, max int) {
	if s.scanCalled {
		panic("Buffer called after Scan")
	}
	s.buf = buf[0:cap(buf)]
	s.maxTokenSize = max
}
```


#### ScanWords
ScanWords按空格分隔进行扫描
```go
func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error) {
	// Skip leading spaces.
	start := 0
	for width := 0; start < len(data); start += width {
		var r rune
		r, width = utf8.DecodeRune(data[start:])
		if !isSpace(r) {
			break
		}
	}
	// Scan until space, marking end of word.
	for width, i := 0, start; i < len(data); i += width {
		var r rune
		r, width = utf8.DecodeRune(data[i:])
		// isSpace 用来检查r是否是个unicode 空格
		if isSpace(r) {
			return i + width, data[start:i], nil
		}
	}
	// If we're at EOF, we have a final, non-empty, non-terminated word. Return it.
	if atEOF && len(data) > start {
		return len(data), data[start:], nil
	}
	// Request more data.
	return start, nil, nil
}

```

#### 自定义SplitFunc
这里我们拿官方的一个Example来说明
```go
func ExampleScanner_custom() {
	// An artificial input source.
	const input = "1234 5678 1234567901234567890"
	scanner := bufio.NewScanner(strings.NewReader(input))
	// Create a custom split function by wrapping the existing ScanWords function.
	split := func(data []byte, atEOF bool) (advance int, token []byte, err error) {
		advance, token, err = bufio.ScanWords(data, atEOF)
		if err == nil && token != nil {
			_, err = strconv.ParseInt(string(token), 10, 32)
		}
		return
	}
	// Set the split function for the scanning operation.
	scanner.Split(split)
	// Validate the input
	for scanner.Scan() {
		fmt.Printf("%s\n", scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		fmt.Printf("Invalid input: %s", err)
	}
	// Output:
	// 1234
	// 5678
	// Invalid input: strconv.ParseInt: parsing "1234567901234567890": value out of range
}
```