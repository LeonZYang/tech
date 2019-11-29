## 深入理解RPC
RPC被称为远程过程调用协议，Go里面也有自带的RPC包，通过这个包我们可以很容易的实现RPC调用

### Gob
将Golang RPC之前我们需要了解下Gob，Gob全称是Go binary，从名字就可以看出来，这个只能在Go里面使用，通过Encode和Decode的方法，我们可以很方便的序列化结构体，一个典型的应用就是net/rpc。具体的例子可以参照官方[https://golang.org/pkg/encoding/gob](https://golang.org/pkg/encoding/gob/)

### RPC服务端
这里我们先介绍下RPC服务端

#### 数据结构

```go
type methodType struct {
	sync.Mutex // protects counters
	method     reflect.Method // 方法
	ArgType    reflect.Type   // 参数
	ReplyType  reflect.Type   // 返回参数
	numCalls   uint           // 调用次数
}

type service struct {
	name   string                 // 服务名称
	rcvr   reflect.Value          // 服务方法接收端
	typ    reflect.Type           // 接收端的类型
	method map[string]*methodType // 注册的方法
}

// Request RPC调用的Header
type Request struct {
	ServiceMethod string   // 方法名称 "Service.Method"
	Seq           uint64   // 序列号，又client产生，自增的
	next          *Request // 下一个空闲的server列表
}

// Response RPC返回钱的Header
type Response struct {
	ServiceMethod string    // 等于Request.ServiceMethod
	Seq           uint64    // 等于Request.Req
	Error         string    // 错误
	next          *Response // 下一个空闲的Server列表
}

// Server represents an RPC Server.
type Server struct {
	serviceMap sync.Map   // map[string]*service
	reqLock    sync.Mutex // protects freeReq
	freeReq    *Request
	respLock   sync.Mutex // protects freeResp
	freeResp   *Response
}

// gobServerCodec 实现了ServerCodec的方法
type gobServerCodec struct {
	rwc    io.ReadWriteCloser
	dec    *gob.Decoder
	enc    *gob.Encoder
	encBuf *bufio.Writer
	closed bool
}


```

#### 注册
```go
// 按照rcvr反射的名称作为ServerName
func (server *Server) Register(rcvr interface{}) error {
	return server.register(rcvr, "", false)
}

// 指定name
func (server *Server) RegisterName(name string, rcvr interface{}) error {
	return server.register(rcvr, name, true)
}
```

##### register
```go
func (server *Server) register(rcvr interface{}, name string, useName bool) error {
    s := new(service) 
    // 获取rvr的type和value
	s.typ = reflect.TypeOf(rcvr)
    s.rcvr = reflect.ValueOf(rcvr)
    // 解析名称，如果useName为true，则使用name
	sname := reflect.Indirect(s.rcvr).Type().Name()
	if useName {
		sname = name
	}
	if sname == "" {
		s := "rpc.Register: no service name for type " + s.typ.String()
		log.Print(s)
		return errors.New(s)
    }
    // 判断是否可导出
	if !isExported(sname) && !useName {
		s := "rpc.Register: type " + sname + " is not exported"
		log.Print(s)
		return errors.New(s)
	}
	s.name = sname

	// 注册方法
	s.method = suitableMethods(s.typ, true)
    // 如果方法为空
	if len(s.method) == 0 {
		str := ""

		// To help the user, see if a pointer receiver would work.
		method := suitableMethods(reflect.PtrTo(s.typ), false)
		if len(method) != 0 {
			str = "rpc.Register: type " + sname + " has no exported methods of suitable type (hint: pass a pointer to value of that type)"
		} else {
			str = "rpc.Register: type " + sname + " has no exported methods of suitable type"
		}
		log.Print(str)
		return errors.New(str)
	}
    // serviceMap 是个sync.Map, 如果名称冲突，则会报错
	if _, dup := server.serviceMap.LoadOrStore(sname, s); dup {
		return errors.New("rpc: service already defined: " + sname)
	}
	return nil
}

```

##### suitableMethods
解析方法，并将其加入到map中
```go
func suitableMethods(typ reflect.Type, reportErr bool) map[string]*methodType {
	methods := make(map[string]*methodType)
	for m := 0; m < typ.NumMethod(); m++ {
		method := typ.Method(m)
		mtype := method.Type
		mname := method.Name
		// 方法必须可以被导出
		if method.PkgPath != "" {
			continue
		}
		// 需要3个输入，rcvr, args, reply
		if mtype.NumIn() != 3 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d input parameters; needs exactly three\n", mname, mtype.NumIn())
			}
			continue
		}
  
        // arg 可以不用是指针
		argType := mtype.In(1)
		if !isExportedOrBuiltinType(argType) {
			if reportErr {
				log.Printf("rpc.Register: argument type of method %q is not exported: %q\n", mname, argType)
			}
			continue
		}
		// 第二个参数(reply)必须是指针
		replyType := mtype.In(2)
		if replyType.Kind() != reflect.Ptr {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not a pointer: %q\n", mname, replyType)
			}
			continue
		}
		// reply类型必须可以被导出
		if !isExportedOrBuiltinType(replyType) {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not exported: %q\n", mname, replyType)
			}
			continue
		}
		// 返回的输出只能有1个
		if mtype.NumOut() != 1 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d output parameters; needs exactly one\n", mname, mtype.NumOut())
			}
			continue
		}
		// 返回的类型必须是error
		if returnType := mtype.Out(0); returnType != typeOfError {
			if reportErr {
				log.Printf("rpc.Register: return type of method %q is %q, must be error\n", mname, returnType)
			}
			continue
		}
		methods[mname] = &methodType{method: method, ArgType: argType, ReplyType: replyType}
	}
	return methods
}
```

##### ServeConn
ServeConn 运行单个连接，阻塞式的知道client连接断开
```go
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),  // decode
		enc:    gob.NewEncoder(buf),   // encode
		encBuf: buf,
	}
	server.ServeCodec(srv)
}
```

##### ServeCodec
ServeCodec 用指定的codec去解码request和编码resposne
```go
func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex)
	wg := new(sync.WaitGroup)
	for {
        // 解码请求
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if debugLog && err != io.EOF {
				log.Println("rpc:", err)
			}
			if !keepReading {
				break
			}
			// 发送response
			if req != nil {
				server.sendResponse(sending, req, invalidRequest, codec, err.Error())
				server.freeRequest(req)
			}
			continue
		}
		wg.Add(1)
		go service.call(server, sending, wg, mtype, req, argv, replyv, codec)
	}
	// We've seen that there are no more requests.
	// Wait for responses to be sent before closing codec.
	wg.Wait()
	codec.Close()
}
```

##### readRequest
readRequest 解码请求
```go
func (server *Server) readRequest(codec ServerCodec) (service *service, mtype *methodType, req *Request, argv, replyv reflect.Value, keepReading bool, err error) {
    // 如果keepReading为true，如果有报错，会将这个request丢弃，然后执行下一个
	service, mtype, req, keepReading, err = server.readRequestHeader(codec)
	if err != nil {
		if !keepReading {
			return
		}
		// discard body
		codec.ReadRequestBody(nil)
		return
	}

	// argv是否是值(非指针)
    argIsValue := false // if true, need to indirect before calling.
    // 将ArgType强制转为指针
	if mtype.ArgType.Kind() == reflect.Ptr {
		argv = reflect.New(mtype.ArgType.Elem())
	} else {
		argv = reflect.New(mtype.ArgType)
		argIsValue = true
	}

    // argv 现在是一个指针
	if err = codec.ReadRequestBody(argv.Interface()); err != nil {
		return
	}
	if argIsValue {
		argv = argv.Elem()
	}

    // ReplyType 肯定是指针
	replyv = reflect.New(mtype.ReplyType.Elem())

	switch mtype.ReplyType.Elem().Kind() {
		// map和slice做初始化
	case reflect.Map:
		replyv.Elem().Set(reflect.MakeMap(mtype.ReplyType.Elem()))
	case reflect.Slice:
		replyv.Elem().Set(reflect.MakeSlice(mtype.ReplyType.Elem(), 0, 0))
	}
	return
}

```

##### readRequestHeader
读取请求header
```go
func (server *Server) readRequestHeader(codec ServerCodec) (svc *service, mtype *methodType, req *Request, keepReading bool, err error) {
	// 新建或者获取空闲的Request
    req = server.getRequest()
    // 解码请求
	err = codec.ReadRequestHeader(req)
	if err != nil {
		req = nil
		if err == io.EOF || err == io.ErrUnexpectedEOF {
			return
		}
		err = errors.New("rpc: server cannot decode request: " + err.Error())
		return
	}

    // 读取成功， 如果有错误，跳过到下一个请求
	keepReading = true

    // 获取最后一个.的索引
	dot := strings.LastIndex(req.ServiceMethod, ".")
	if dot < 0 {
		err = errors.New("rpc: service/method request ill-formed: " + req.ServiceMethod)
		return
    }
    // 服务名称
    serviceName := req.ServiceMethod[:dot]
    // 方法名称
	methodName := req.ServiceMethod[dot+1:]

    // 获取请求
	svci, ok := server.serviceMap.Load(serviceName)
	if !ok {
		err = errors.New("rpc: can't find service " + req.ServiceMethod)
		return
	}
    svc = svci.(*service)
    // 获取methodType
	mtype = svc.method[methodName]
	if mtype == nil {
		err = errors.New("rpc: can't find method " + req.ServiceMethod)
	}
	return
}
```

##### call
调用
```go
func (s *service) call(server *Server, sending *sync.Mutex, wg *sync.WaitGroup, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
	if wg != nil {
		defer wg.Done()
	}
	mtype.Lock()
	mtype.numCalls++
	mtype.Unlock()
	function := mtype.method.Func
	// Invoke the method, providing a new value for the reply.
	returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
	// 返回的是个error
	errInter := returnValues[0].Interface()
	errmsg := ""
	if errInter != nil {
		errmsg = errInter.(error).Error()
    }
    // 发送响应
	server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
	server.freeRequest(req)
}
```

##### sendResponse
发送响应
```go
func (server *Server) sendResponse(sending *sync.Mutex, req *Request, reply interface{}, codec ServerCodec, errmsg string) {
    // 新建或者获取个response
	resp := server.getResponse()

	resp.ServiceMethod = req.ServiceMethod
	if errmsg != "" {
		resp.Error = errmsg
		reply = invalidRequest
	}
	resp.Seq = req.Seq
    sending.Lock()
    // 编码response, gobServerCodec.WriteResponse
	err := codec.WriteResponse(resp, reply)
	if debugLog && err != nil {
		log.Println("rpc: writing response:", err)
	}
    sending.Unlock()
    // 将freeResponse放回freeList
	server.freeResponse(resp)
}
```


#### gobServerCodec
gobServerCodec 是gob的编码和解码结构
```go
type gobServerCodec struct {
	rwc    io.ReadWriteCloser
	dec    *gob.Decoder
	enc    *gob.Encoder
	encBuf *bufio.Writer
	closed bool
}

func (c *gobServerCodec) ReadRequestHeader(r *Request) error {
	return c.dec.Decode(r)
}

func (c *gobServerCodec) ReadRequestBody(body interface{}) error {
	return c.dec.Decode(body)
}

func (c *gobServerCodec) WriteResponse(r *Response, body interface{}) (err error) {
	if err = c.enc.Encode(r); err != nil {
		if c.encBuf.Flush() == nil {
			// Gob couldn't encode the header. Should not happen, so if it does,
			// shut down the connection to signal that the connection is broken.
			log.Println("rpc: gob error encoding response:", err)
			c.Close()
		}
		return
	}
	if err = c.enc.Encode(body); err != nil {
		if c.encBuf.Flush() == nil {
			// Was a gob problem encoding the body but the header has been written.
			// Shut down the connection to signal that the connection is broken.
			log.Println("rpc: gob error encoding body:", err)
			c.Close()
		}
		return
	}
	return c.encBuf.Flush()
}

func (c *gobServerCodec) Close() error {
	if c.closed {
		// Only call c.rwc.Close once; otherwise the semantics are undefined.
		return nil
	}
	c.closed = true
	return c.rwc.Close()
}

```


### Client
客户端源码

#### 数据结构
```go
type Call struct {
	ServiceMethod string      // 调用的服务名和方法名
	Args          interface{} // 参数
	Reply         interface{} // 返回值
	Error         error       // 调用结束后的Error
	Done          chan *Call  // 调用结束时关闭
}

type Client struct {
	codec ClientCodec

	reqMutex sync.Mutex // protects following
	request  Request

	mutex    sync.Mutex // protects following
	seq      uint64     // 自增
	pending  map[uint64]*Call  // key是seq
	closing  bool // 是否close
	shutdown bool // 服务是否关闭
}

type ClientCodec interface {
	WriteRequest(*Request, interface{}) error
	ReadResponseHeader(*Response) error
	ReadResponseBody(interface{}) error

	Close() error
}
```

#### NewClient
```go
func NewClient(conn io.ReadWriteCloser) *Client {
	encBuf := bufio.NewWriter(conn)
	client := &gobClientCodec{conn, gob.NewDecoder(conn), gob.NewEncoder(encBuf), encBuf}
	return NewClientWithCodec(client)
}

// NewClientWithCodec is like NewClient but uses the specified
// codec to encode requests and decode responses.
func NewClientWithCodec(codec ClientCodec) *Client {
	client := &Client{
		codec:   codec,
		pending: make(map[uint64]*Call),
	}
	go client.input()
	return client
}

```

##### input
```go
func (client *Client) input() {
	var err error
	var response Response
	for err == nil {
        response = Response{}
        // 读取响应头
		err = client.codec.ReadResponseHeader(&response)
		if err != nil {
			break
		}
		seq := response.Seq
        client.mutex.Lock()
        // 根据seq获取对应的Call
        call := client.pending[seq]
        // 删除对应的key
		delete(client.pending, seq)
		client.mutex.Unlock()

		switch {
        case call == nil:
            // 没有等待响应的请求，这种情况下WriteRequest请求部分失败，call已经被删除，这种情况下我们仍然需要读取错误的body
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
		case response.Error != "":
            // 捕获错误
			call.Error = ServerError(response.Error)
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
            }
            // 结束
			call.done()
        default:
            // 解码reply
			err = client.codec.ReadResponseBody(call.Reply)
			if err != nil {
				call.Error = errors.New("reading body " + err.Error())
            }
            // 结束
			call.done()
		}
	}

    // 将等待调用的停止
	client.reqMutex.Lock()
	client.mutex.Lock()
	client.shutdown = true
	closing := client.closing
	if err == io.EOF {
		if closing {
			err = ErrShutdown
		} else {
			err = io.ErrUnexpectedEOF
		}
    }  
    // 批量结束
	for _, call := range client.pending {
		call.Error = err
		call.done()
	}
	client.mutex.Unlock()
	client.reqMutex.Unlock()
	if debugLog && err != io.EOF && !closing {
		log.Println("rpc: client protocol error:", err)
	}
}
```

##### Go
主要调用，Go是个异步调用
```go
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call {
	call := new(Call)
	call.ServiceMethod = serviceMethod
	call.Args = args
	call.Reply = reply
	// done 这里不要使用无缓存的chan
	if done == nil {
        // 有缓存的chan
		done = make(chan *Call, 10) // buffered.
	} else {
        // 如果done不为nil, done必须有足够的缓存去容纳同时发生RPC调用
		if cap(done) == 0 {
			log.Panic("rpc: done channel is unbuffered")
		}
	}
	call.Done = done
	client.send(call)
	return call
}

// call是个同步调用
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}
```

##### send
send 发送请求
```go
func (client *Client) send(call *Call) {
	client.reqMutex.Lock()
	defer client.reqMutex.Unlock()

	// 注册
	client.mutex.Lock()
	if client.shutdown || client.closing {
		client.mutex.Unlock()
		call.Error = ErrShutdown
		call.done()
		return
	}
	seq := client.seq
	client.seq++
	client.pending[seq] = call
	client.mutex.Unlock()

	// 编码发送seq
	client.request.Seq = seq
	client.request.ServiceMethod = call.ServiceMethod
	err := client.codec.WriteRequest(&client.request, call.Args)
	if err != nil {
		client.mutex.Lock()
		call = client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()
		if call != nil {
			call.Error = err
			call.done()
		}
	}
}
```

#### gobClientCodec

```go
type gobClientCodec struct {
	rwc    io.ReadWriteCloser
	dec    *gob.Decoder
	enc    *gob.Encoder
	encBuf *bufio.Writer
}

func (c *gobClientCodec) WriteRequest(r *Request, body interface{}) (err error) {
	if err = c.enc.Encode(r); err != nil {
		return
	}
	if err = c.enc.Encode(body); err != nil {
		return
	}
	return c.encBuf.Flush()
}

func (c *gobClientCodec) ReadResponseHeader(r *Response) error {
	return c.dec.Decode(r)
}

func (c *gobClientCodec) ReadResponseBody(body interface{}) error {
	return c.dec.Decode(body)
}

func (c *gobClientCodec) Close() error {
	return c.rwc.Close()
}

```


总结：官方自带的RPC还是比较简单的，目前RPC不再支持新功能，可以的话，还是使用gRPC
1. Go里面rpc client调用没有实现超时的控制，需要我们自己实现下超时。
2. RPC Server没有实现存活检测，如果遇到网络异常情况下，可能会服务端会产生很多无用连接(ESTABLISH)