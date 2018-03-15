### 1. rpc/server.go

![image](https://raw.githubusercontent.com/fengyfei/wizard/b340cd33f115c9bf41dd80bd8aee92c129191f53/go/rpc/images/rpc_server.png?token=AXUDI6eLePjfKv4GGm7-ZyJlpltPlVzDks5asd7bwA%3D%3D)

```go
type service struct {
	name   string                 // 服务名
	rcvr   reflect.Value          // 服务中函数的接收者
	typ    reflect.Type           // 接收者类型
	method map[string]*methodType // 已注册的函数集
}

// RPC Server
type Server struct {
	serviceMap sync.Map   // 服务对象集合
	reqLock    sync.Mutex // 请求锁用来保护 freeReq
	freeReq    *Request
	respLock   sync.Mutex // 响应锁保护 freeResp
	freeResp   *Response
}

// Register 在 server 中注册并发布 receiver 的函数集时需满足以下条件:
//  * 函数和函数的类型名是已导出的
//  * 两个参数都是导出类型(或內建类型)
//  * 第二个参数是指针
//  * 函数只有一个类型为 error 的返回类型
// 如果 receiver 不是导出的类型或者没有符合条件的函数，将会返回一个错误
// Register 将会使用 log 包记录出现的 error
// 客户端使用 "Type.Method" 的格式来调用函数，比如上文例子中 Arith.Multiply
// 这里的 Type 是 receiver 的具体类型.
func (server *Server) register(rcvr interface{}, name string, useName bool) error {
	...

	// 判断传入的接口对象的函数是否符合 RPC 规范
	s.method = suitableMethods(s.typ, true)

	...

    // LoadOrStore 会检查 sync.Map 类型对象中是否存在传入的键名，如果存在则返回相应的值和 true
    // 反之会先存入键值对再返回值和 false
	if _, dup := server.serviceMap.LoadOrStore(sname, s); dup {
		return errors.New("rpc: service already defined: " + sname)
	}
	...
}

// 客户端请求后，服务端通过 call 调用相应服务
func (s *service) call(server *Server, sending *sync.Mutex, wg *sync.WaitGroup, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
	if wg != nil {
		defer wg.Done()
	}
	mtype.Lock()
	mtype.numCalls++
	mtype.Unlock()
	function := mtype.method.Func
	// 执行函数, 返回新的值给 reply
	returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
	// 返回值里的错误
	errInter := returnValues[0].Interface()
	errmsg := ""
	if errInter != nil {
		errmsg = errInter.(error).Error()
	}
	server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
	server.freeRequest(req)
}

// 此处使用 gob 包来进行序列化工作，jsonRPC 定义的编解码器使用"encoding/json"代替 gob
type gobServerCodec struct {
	rwc    io.ReadWriteCloser
	dec    *gob.Decoder
	enc    *gob.Encoder
	encBuf *bufio.Writer
	closed bool
}

// 客户端和服务端在发送数据前都进行了编码，使用前都需要解码
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
	// *bufio.Writer.Flush() 将已 buffered 的数据写入内部的 io.writer
}

// ServeConn 在一个单连接上运行 server
// ServeConn 阻塞, 在服务该连接到客户端挂起的期间.
// 一般另起线程来调用本函数，比如 `go server.ServeConn(conn)` (Accept 函数中有调用)
// ServeConn 在该连接上使用 gob 包的有线格式 (参见 gob 包) .
// 如需使用其他备份 编解码器, 使用 ServeCodec 函数.
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),
		enc:    gob.NewEncoder(buf),
		encBuf: buf,
	}
	server.ServeCodec(srv)
}

// ServeCodec 与 ServeConn 类似， 除了使用指定的编解码器来解码 requests 和编码 responses
func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex)
	wg := new(sync.WaitGroup)
	for {
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if debugLog && err != io.EOF {
				log.Println("rpc:", err)
			}
			if !keepReading {
				break
			}
			// send a response if we actually managed to read a header.
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

// Accept 从监听器上接收获取到的连接并服务每个连接的请求
// Accept 阻塞直到监听器返回非空的错误
// 调用者一般应使用 goroutine 启用 Accept，比如 `go server.Accept(l)`
func (server *Server) Accept(lis net.Listener) {
	for {
		conn, err := lis.Accept()
		if err != nil {
			log.Print("rpc.Serve: accept:", err.Error())
			return
		}
		go server.ServeConn(conn)
	}
}

// ServeHTTP 实现一个用于回应 RPC 请求的 http.Handler
func (server *Server) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if req.Method != "CONNECT" {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.WriteHeader(http.StatusMethodNotAllowed)
		io.WriteString(w, "405 must CONNECT\n")
		return
	}
	conn, _, err := w.(http.Hijacker).Hijack() // 让调用者主动接管连接
	if err != nil {
		log.Print("rpc hijacking ", req.RemoteAddr, ": ", err.Error())
		return
	}
	io.WriteString(conn, "HTTP/1.0 "+connected+"\n\n")
	server.ServeConn(conn)
}

// HandleHTTP 注册 server 的 RPC 信息到 rpcPath 上，注册 server 的 debug 信息到 debugPath 上
// HandleHTTP 会注册到 http.DefaultServeMux上
// 之后，仍需要调用 http.Serve()，一般会另起线程："go http.Serve(l, nil)"
func (server *Server) HandleHTTP(rpcPath, debugPath string) {
	http.Handle(rpcPath, server)
	http.Handle(debugPath, debugHTTP{server})
}
```

### 2. rpc/client.go

![image](https://raw.githubusercontent.com/fengyfei/wizard/b340cd33f115c9bf41dd80bd8aee92c129191f53/go/rpc/images/rpc_client.png?token=AXUDI5t9jN7S-y8pQTFycid-jcWBAKtQks5asd5twA%3D%3D)

```go
// Call 代表一个活跃的 RPC.
type Call struct {
	ServiceMethod string      // 调用的服务名
	Args          interface{} // 函数传入参数 (*struct)
	Reply         interface{} // 函数返回结果 (*struct)
	Error         error       // 结束后的错误状态
	Done          chan *Call  // Strobes when call is complete.
}

// Client 代表一个 RPC 客户端
// 同一个客户端可能有多个未返回的调用，也可能被多个 go 线程同时使用
type Client struct {
	codec ClientCodec
	reqMutex sync.Mutex // 保护 request
	request  Request
	mutex    sync.Mutex // 保护 seq
	seq      uint64
	pending  map[uint64]*Call
	closing  bool // 用户已调用 Close
	shutdown bool // 服务器已告知停止
}

// ClientCodec 接口实现了 RPC 会话的客户端一侧 RPC 请求的写入和 RPC 响应的读取。
// 客户端调用 WriteRequest 来写入请求到连接，然后成对调用 ReadRsponseHeader 和
// ReadResponseBody 以读取响应。客户端在结束该连接的事务时调用 Close 方法。
// ReadResponseBody 可以使用 nil 参数调用，以强制回复的主体被读取然后丢弃。
type ClientCodec interface {
	// WriteRequest 必须能安全的被多个go程同时使用
	WriteRequest(*Request, interface{}) error
	ReadResponseHeader(*Response) error
	ReadResponseBody(interface{}) error

	Close() error
}

// 以一个死循环的方式不断地从连接中读取 response, 然后调用 map 中读取等待的 Call.Done 的 channel 通知完成
func (client *Client) input() {
	var err error
	var response Response
	for err == nil {
		response = Response{}
		err = client.codec.ReadResponseHeader(&response)
		if err != nil {
			break
		}
		seq := response.Seq
		client.mutex.Lock()
		call := client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()

		switch {
		case call == nil:
			// 无等待中的 call. 一般意味着 WriteRequest 失败了并且 call 已被去除
             // response is a server telling us about an
			// error reading request body. We should still attempt
			// to read error body, but there's no one to give it to.
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
		case response.Error != "":
			// 获取到一个错误响应. 将这个传给 request;
			// 任何后续的 requests 都会获取到 ReadResponseBody error，如果有的话
			call.Error = ServerError(response.Error)
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
			call.done()
		default:
			err = client.codec.ReadResponseBody(call.Reply)
			if err != nil {
				call.Error = errors.New("reading body " + err.Error())
			}
			call.done()
		}
	}
	// 关闭等待中的 calls.
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

// NewClient 返回一个新的 Client，以管理对连接另一端的服务的请求。
// 它添加缓冲到连接的写入侧，以便将回复的头域和有效负载作为一个单元发送。
func NewClient(conn io.ReadWriteCloser) *Client {
	encBuf := bufio.NewWriter(conn)
	client := &gobClientCodec{conn, gob.NewDecoder(conn), gob.NewEncoder(encBuf), encBuf}
	return NewClientWithCodec(client)
}

// DialHTTP 通过地址连向一个 HTTP RPC server
func DialHTTP(network, address string) (*Client, error) {
	return DialHTTPPath(network, address, DefaultRPCPath)
}

// DialHTTPPath 通过地址和路径连向一个 HTTP RPC server
func DialHTTPPath(network, address, path string) (*Client, error) {
	var err error
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	io.WriteString(conn, "CONNECT "+path+" HTTP/1.0\n\n")

	// 在切换 RPC 协议前需要保证成功的 HTTP 响应
	resp, err := http.ReadResponse(bufio.NewReader(conn), &http.Request{Method: "CONNECT"})
	if err == nil && resp.Status == connected {
		return NewClient(conn), nil
	}
	if err == nil {
		err = errors.New("unexpected HTTP response: " + resp.Status)
	}
	conn.Close()
	return nil, &net.OpError{
		Op:   "dial-http",
		Net:  network + " " + address,
		Addr: nil,
		Err:  err,
	}
}

// Dial 通过指定的地址连向一个 RPC server
func Dial(network, address string) (*Client, error) {
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	return NewClient(conn), nil
}

// Go 异步地执行函数. 本方法 Call 结构体类型指针的返回值代表该次远程调用. 
// 通道类型的参数 done 会在本次调用完成时发出信号（通过返回本次Go方法的返回值）
// 如果 done 为nil，Go 会申请一个新的通道（写入返回值的 Done 字段）
// 如果 done 非nil，done 必须有缓冲，否则 Go 方法会故意崩溃。
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call {
	call := new(Call)
	call.ServiceMethod = serviceMethod
	call.Args = args
	call.Reply = reply
	if done == nil {
		done = make(chan *Call, 10) // buffered.
	} else {
		// 如果调用者传的 done != nil，则必须确保通道有足够的缓冲来给多个同步 RPCs 使用
		// 如果通道完全没有缓冲，最好不要去运行
		if cap(done) == 0 {
			log.Panic("rpc: done channel is unbuffered")
		}
	}
	call.Done = done
	client.send(call)
	return call
}

// Call 调用传入名的远程服务，并等待结束返回结果和错误状态
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error {
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}
```

### 3. net/rpc/jsonrpc

jsonrpc 主要将 gob 序列化工具换成 json 序列化工具，主要函数还是调用 server 里的 FuncWithCodec 函数，原理基本一致


### References

- [Go RPC 开发指南](https://www.gitbook.com/book/smallnest/go-rpc-programming-guide/details)
- [Go 官方库 RPC 开发指南](http://colobu.com/2016/09/18/go-net-rpc-guide/) 
- [build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/08.4.md)
- [rpc wikipedia](https://en.wikipedia.org/wiki/Remote_procedure_call)
- [How RPC Works](https://technet.microsoft.com/en-us/library/cc738291%28v=ws.10%29.aspx?f=255&MSPPError=-2147217396)
