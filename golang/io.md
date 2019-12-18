## 深入理解IO

### 接口

#### Reader
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
Read读取len(p)个字节到p中，返回的读取到字节的数量(0<=n<=len(p))和发生的错误。
及时Read返回的n < len(p), 它也会在调用过程中暂用len(p)个字节做为暂存空间。如果读取到的数据小于len(p)个字节，Read会返回可用的数据，而不是等待更多数据。
当Read在成功读取n>0个字节后遇到一个错误或者EOF，它会返回读取到的字节数。它可能会同时在本次调用中返回一个non-nil的错误或是在下一次调用中返回这个错误(且n=0)。一般情况下，Reader读取到结尾会返回一个非0的字节数量，可能err返回EOF或者nil，并且之后的Read会返回(n=0, err=EOF)

调用方应该在处理错误前处理n>0的字节数据，这样做可以正确处理在读取字节后产生的IO错误，同时也允许EOF的出现

实现Read应该避免返回n=0和err=nil，除非len(p)=0, 同时返回n=0和err=nil，将表示什么都没有发生。

#### Writer
```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```
Writer实现了write方法，write写入len(p)个字节， 返回的字节0<=n<=len(p)和导致写入失败的错误。
Write如果返回的n小于len(p)，必须返回个non-nil错误。Write不允许修改p，即便是暂时也不行。


#### 实现io.Reader或是io.Writer接口的类型

* `os.File` 同时实现了 `io.Reader` 和 `io.Writer`
* `strings.Reader` 实现了 `io.Reader`
* `bufio.Reader/Writer` 分别实现了 `io.Reader` 和 `io.Writer`
* `bytes.Buffer` 同时实现了 `io.Reader` 和 `io.Writer`
* `bytes.Reader` 实现了 `io.Reader`
* `compress/gzip.Reader/Writer` 分别实现了 `io.Reader` 和 `io.Writer`
* `crypto/cipher.StreamReader/StreamWriter` 分别实现了 `io.Reader` 和 `io.Writer`
* `crypto/tls.Conn` 同时实现了 `io.Reader` 和 `io.Writer`
* `encoding/csv.Reader/Writer` 分别实现了 `io.Reader` 和 `io.Writer`
* `mime/multipart.Part` 实现了 `io.Reader`
* `io.LimitedReader`、`io.PipeReader`、`io.SectionReader`实现了`io.Reader`
* `io.PipeWriter`实现了`io.Writer`

大家可以看下我的另一篇文章`深入理解bufio`

#### ReaderAt和WriterAt
```go
type ReaderAt interface {
	ReadAt(p []byte, off int64) (n int, err error)
}
```
ReaderAt从输入源的偏移量off读取len(p)个字节，它会返回读取的字节(0<=n<=len(p))和任何遇到的错误。
如果ReadAt返回的n < len(p)，它返回一个non-nil的错误来解释为什么没有返回更多的字节，这一点上，ReadAt比Read更严格。
及时ReadAt返回的n < len(p)，它也会使用p做全部的暂存空间。若可读取的数据不到len(p)字节，ReadAt就会阻塞知道所有的数据都可用或一个错误发生，这点ReadAt不同于Read。
如果n == len(p) 个字节从输入源的结尾由ReadAt返回，Read可能返回err == EOF或者err == nil。
如果ReadAt读取一个带偏移量的输入源，ReadAt不应该影响它或者被它影响
可对相同的数据源执行并发调用

```
type WriterAt interface {
	WriteAt(p []byte, off int64) (n int, err error)
}
```
WriteAt从p中将len(p)个字节写入到偏移量off的数据流中。它返回从p中被写入的字节数n(0 <= n <=len(p)) 以及任何遇到的引起写入停止的错误。若WriteAt返回的n < len(p)，它就必须返回一个non-nil的错误
如果WriteAt写入一个带有偏移量的源，它不应该影响或者被它影响（和ReadAt一样）
WriteAt允许对于不同的区域进行并发写入

#### Copy
```go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```
Copy拷贝从src到dst直到在src遇到EOF或者发生错误，它返回拷贝的字节数量和拷贝过程中发生的第一个错误。
一个成功的copy返回的err == nil,并不是err == EOF。因为copy的定义就是从src拷贝直到EOF，它不会将这个定义为错误。
如果src实现了WriterTo接口，那么copy就会调用src.WriteTo(dst)，否则如果dst实现了ReaderFrom接口，那么就会调用dst.ReadFrom(src)。

##### copyBuffer
如果buf == nil，那么就会分配一个
```go
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		// 如果src是LimitedReader，则将buf的大小设置为LimitedReader.N
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw > 0 {
				written += int64(nw)
			}
			if ew != nil {
				err = ew
				break
			}
			// 写入和读取的字节不一致
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```


##### LimitReader
```go
// LimitReader returns a Reader that reads from r
// but stops with EOF after n bytes.
// The underlying implementation is a *LimitedReader.
func LimitReader(r Reader, n int64) Reader { return &LimitedReader{r, n} }

// A LimitedReader reads from R but limits the amount of
// data returned to just N bytes. Each call to Read
// updates N to reflect the new amount remaining.
// Read returns EOF when N <= 0 or when the underlying R returns EOF.
type LimitedReader struct {
	R Reader // underlying reader
	N int64  // max bytes remaining
}

func (l *LimitedReader) Read(p []byte) (n int, err error) {
	if l.N <= 0 {
		return 0, EOF
	}
	if int64(len(p)) > l.N {
		p = p[0:l.N]
	}
	n, err = l.R.Read(p)
	l.N -= int64(n)
	return
}
```


##### Pipe
Pipe创建了一个同步的内存管道（没有内部缓冲），可以进行一对一、一对多、多对一，多对多的数据传输。 并发写和并发读是安全的。
除非有read，否则write是阻塞式的写入
```go
func Pipe() (*PipeReader, *PipeWriter) {
	// 都是无缓存的chan
	p := &pipe{
		wrCh: make(chan []byte),
		rdCh: make(chan int),
		done: make(chan struct{}),
	}
	return &PipeReader{p}, &PipeWriter{p}
}
```
PipeReader和PipeWriter也是使用pipe结构进行交互

```go
type PipeReader struct {
	p *pipe
}
type PipeWriter struct {
	p *pipe
}
```

###### pipe
pipe是PipeReader和PipeWriter共享管道结构
```go
type pipe struct {
	wrMu sync.Mutex // Serializes Write operations
	wrCh chan []byte
	rdCh chan int

	once sync.Once // Protects closing done
	done chan struct{}
	rerr atomicError
	werr atomicError
}
```

##### Read
```go
func (p *pipe) Read(b []byte) (n int, err error) {
	select {
	case <-p.done:
		return 0, p.readCloseError()
	default:
	}

	select {
	case bw := <-p.wrCh:     // 从wrCh中读取数据
		nr := copy(b, bw)    // 将读取的数据copy到b中
		p.rdCh <- nr         // 将读取的字节数量写入到rdCh，这个在write里说下
		return nr, nil       // 返回读取到字节的数量和错误
	case <-p.done:
		return 0, p.readCloseError()
	}
}
```

##### Write
```go
func (p *pipe) Write(b []byte) (n int, err error) {
	select {
	case <-p.done:
		return 0, p.writeCloseError()
	default:
		// 加锁，保证写入顺序
		p.wrMu.Lock()
		defer p.wrMu.Unlock()
	}

	// once主要的作用是防止当len(b) == 0 ，不会进行此判断，这样就无法和Read沟通
	for once := true; once || len(b) > 0; once = false {
		select {
		case p.wrCh <- b:    // 将数据写入到wrCh
			nw := <-p.rdCh   // 从rdCh同步上一次读取的数量（当Read一次没读取完，下次Read需要继续读取）
			b = b[nw:]       // 修改b
			n += nw          // 累加写入的字节数量
		case <-p.done:
			return n, p.writeCloseError()
		}
	}
	return n, nil
}
```