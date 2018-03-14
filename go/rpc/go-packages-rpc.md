### 1. 定义

RPC（Remote Procedure Call Protocol）——远程过程调用协议，是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。它假定某些传输协议的存在，如TCP或UDP，以便为通信程序之间携带信息数据。通过它可以使函数调用模式网络化。在[OSI网络通信模型](https://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B)中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。

![image](https://raw.githubusercontent.com/wangriyu/books/master/%E5%90%8E%E5%8F%B0BackEnd/rpc-def.png)

### 2. 工作原理

![image](https://img.ctolib.com/uploadImg/20161108/20161108125553_878.png)

流程：
1. 调用客户端句柄；执行传送参数
2. 调用本地系统内核发送网络消息
3. 消息传送到远程主机
4. 服务器句柄得到消息并取得参数
5. 执行远程过程
6. 执行的过程将结果返回服务器句柄
7. 服务器句柄返回结果，调用远程系统内核
8. 消息传回本地主机
9. 客户句柄由内核接收消息
10. 客户接收句柄返回的数据

### 3. Go RPC

Go标准包中已经提供了对RPC的支持，而且支持三个级别的RPC：TCP、HTTP、JSONRPC。但Go的RPC包是独一无二的RPC，它和传统的RPC系统不同，它只支持Go开发的服务器与客户端之间的交互，因为在内部，它们采用了Gob来编码。

任何的RPC都需要通过网络来传递数据，Go RPC可以利用HTTP和TCP来传递数据，利用HTTP的好处是可以直接复用net/http里面的一些函数。

rpc包提供了通过网络或其他I/O连接对一个对象的导出函数的访问。服务端注册一个对象，使它作为一个服务被暴露，服务的名字是该对象的类型名。注册之后，对象的导出函数就可以被远程访问。服务端可以注册多个不同类型的对象（服务），但注册具有相同类型的多个对象是错误的。

函数只有符合下面的条件才能被远程访问，不然会被忽略，详细的要求如下：

- 函数必须是导出的(首字母大写)，函数的类型名也是导出的
- 必须有两个导出类型(或內建类型)的参数，
  第一个参数是接收的参数，第二个参数是返回给客户端的参数，第二个参数必须是指针类型的
- 函数只有一个error接口类型的返回值

举个例子，正确的RPC函数格式如下：

```
func (t *T) MethodName(argType T1, replyType *T2) error
```
**T、T1和T2类型必须能被encoding/gob包编解码**, 这些限制即使使用不同的编解码器也适用。（未来，对定制的编解码器可能会使用较宽松一点的限制）
​	
方法的第一个参数代表调用者提供的参数；第二个参数代表返回给调用者的参数。方法的返回值，如果非nil，将被作为字符串回传，在客户端看来就和errors.New创建的一样。如果返回了错误，回复的参数将不会被发送给客户端。

服务端可能会单个连接上调用ServeConn管理请求。更典型地，它会创建一个网络监听器然后调用Accept；或者，对于HTTP监听器，调用HandleHTTP和http.Serve。

想要使用服务的客户端会创建一个连接，然后用该连接调用NewClient。

更方便的函数Dial（DialHTTP）会在一个原始的连接（或HTTP连接）上依次执行这两个步骤。
生成的Client类型值有两个方法，Call和Go，它们的参数为要调用的服务和方法、一个包含参数的指针、一个用于接收接个的指针。

Call方法会等待远端调用完成，而Go方法异步的发送调用请求并使用返回的Call结构体类型的Done通道字段传递完成信号。

除非设置了显式的编解码器，本包默认使用encoding/gob包来传输数据。

这是一个简单的例子。一个服务端想要导出Arith类型的一个对象：

```go
package server

import "errors"

type Args struct {
	A, B int
}

type Quotient struct {
	Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("divide by zero")
	}
    quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}
```

服务端会调用（用于HTTP服务）：
```go
arith := new(Arith)
rpc.Register(arith)
rpc.HandleHTTP()
l, e := net.Listen("tcp", ":1234")
checkError(err)
go http.Serve(l, nil)
```
此时，客户端可看到服务"Arith"及它的方法"Arith.Multiply"、"Arith.Divide"。要调用方法，客户端首先呼叫服务端：

```go
client, err := rpc.DialHTTP("tcp", serverAddress + ":1234")
checkError(err)
```
然后，客户端可以执行远程调用：

```go
// 同步调用
args := &server.Args{7,8}
var reply int
err = client.Call("Arith.Multiply", args, &reply)
checkError(err)
fmt.Printf("Arith: %d*%d=%d", args.A, args.B, reply)
```
或者

```go
// 异步调用
quotient := new(Quotient)
divCall := client.Go("Arith.Divide", args, quotient, nil)
replyCall := <-divCall.Done	// will be equal to divCall
// check errors, print, etc.
```
另外 TCP 方式下的服务端及客户端代码：

```
// server
arith := new(Arith)
rpc.Register(arith)

tcpAddr, err := net.ResolveTCPAddr("tcp", ":1234")
checkError(err)

listener, err := net.ListenTCP("tcp", tcpAddr)
checkError(err)

// 手动监听请求并调用 ServeConn
for {
	conn, err := listener.Accept()
	if err != nil {
	continue
}

rpc.ServeConn(conn)

// client
client, err := rpc.Dial("tcp", service) // 此处调用 Dial 函数
checkError(err)

args := &server.Args{17, 8}
var reply int
err = client.Call("Arith.Multiply", args, &reply)
if err != nil {
	log.Fatal("arith error:", err)
}
fmt.Printf("Arith: %d*%d=%d\n", args.A, args.B, reply)
```
jsonRPC方式下的服务端及客户端代码：

```
import "net/rpc/jsonrpc"

// server
arith := new(Arith)
rpc.Register(arith)

tcpAddr, err := net.ResolveTCPAddr("tcp", ":1234")
checkError(err)

listener, err := net.ListenTCP("tcp", tcpAddr)
checkError(err)

for {
	conn, err := listener.Accept()
	if err != nil {
		continue
	}
	jsonrpc.ServeConn(conn) // 使用 jaonrpc
}

// client
client, err := jsonrpc.Dial("tcp", service) // 使用 jaonrpc
checkError(err)

args := Args{17, 8}
var reply int
err = client.Call("Arith.Multiply", args, &reply)
if err != nil {
	log.Fatal("arith error:", err)
}
fmt.Printf("Arith: %d*%d=%d\n", args.A, args.B, reply)
```
jsonRPC 是基于TCP协议实现的，目前它还不支持HTTP方式

服务端的实现应为客户端提供简单、类型安全的包装。

net/rpc 包不会再接受新特性，如果需要多语言支持或高可用性可以使用第三方 RPC 库:

- **grpc-go**[https://github.com/grpc/grpc-go]: The Go implementation of gRPC: A high performance, open source, general RPC framework that puts mobile and HTTP/2 first.
- **go-micro**[https://github.com/micro/go-micro]: Go Micro is a pluggable RPC framework for distributed systems development. 
- **thrift**[https://github.com/apache/thrift]: Thrift is a lightweight, language-independent software stack with an associated code generation mechanism for RPC. 

---

# net/rpc源码:

### 1. rpc/server.go

![image](https://raw.githubusercontent.com/fengyfei/wizard/d16dc5a4e3a33bdfd2b37175355a990d83a6f873/go/rpc/images/rpc_server.png?token=AXUDIxcqXswM3eg0vztZTMIPXt0w_ycuks5ascxzwA%3D%3D)

```go
type methodType struct {
	sync.Mutex // protects counters
	method     reflect.Method
	ArgType    reflect.Type
	ReplyType  reflect.Type
	numCalls   uint
}

type service struct {
	name   string                 // 服务名
	rcvr   reflect.Value          // 服务中函数的接收者
	typ    reflect.Type           // 接收者类型
	method map[string]*methodType // 已注册的函数集
}

// Request 是每个 RPC 调用请求的 Header。它是被内部使用的，这里的文档用于帮助 debug，如分析网络拥堵时。
type Request struct {
	ServiceMethod string   // 格式："Service.Method"
	Seq           uint64   // 客户端选择的序列号
	next          *Request // for free list in Server
}

// Response 是每个RPC 调用回复的 Header。它是被内部使用的，这里的文档用于帮助debug，如分析网络拥堵时。
type Response struct {
	ServiceMethod string    // 对应请求的同一字段
	Seq           uint64    // 对应请求的同一字段
	Error         string    // 可能的错误
	next          *Response // for free list in Server
}

// RPC Server
type Server struct {
	serviceMap sync.Map   // 服务对象集合
	reqLock    sync.Mutex // 请求锁用来保护 freeReq
	freeReq    *Request
	respLock   sync.Mutex // 响应锁保护 freeResp
	freeResp   *Response
}

// 创建一个新 server
func NewServer() *Server {
	return &Server{}
}

// DefaultServer 是 *Server 的默认实例
var DefaultServer = NewServer()

// 判断输入名是否是已导出的
func isExported(name string) bool {...}

// 判断输入类型是否是已导出的或內建类型
func isExportedOrBuiltinType(t reflect.Type) bool {...}

// Register 在 server 中注册并发布 receiver 的函数集时需满足以下条件:
//  * 函数和函数的类型名是已导出的
//  * 两个参数都是导出类型(或內建类型)
//  * 第二个参数是指针
//  * 函数只有一个类型为 error 的返回类型
// 如果 receiver 不是导出的类型或者没有符合条件的函数，将会返回一个错误
// Register 将会使用 log 包记录出现的 error
// 客户端使用 "Type.Method" 的格式来调用函数，比如上文例子中 Arith.Multiply
// 这里的 Type 是 receiver 的具体类型.
func (server *Server) Register(rcvr interface{}) error {
	return server.register(rcvr, "", false)
}

// RegisterName 指定了一个名字作为服务名，客户端调用函数集时使用这个名字来代替原来 receiver 的具体类型
func (server *Server) RegisterName(name string, rcvr interface{}) error {
	return server.register(rcvr, name, true)
}

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

// suitableMethods 返回 typ 的符合条件的函数集
// 如果 reportErr 为 true, 它会使用 log 包记录 error 日志
func suitableMethods(typ reflect.Type, reportErr bool) map[string]*methodType {
	methods := make(map[string]*methodType)
	for m := 0; m < typ.NumMethod(); m++ {
		method := typ.Method(m)
		mtype := method.Type
		mname := method.Name
		// 函数必须是已导出的
		if method.PkgPath != "" {
			continue
		}
		// 函数需要三个元素: receiver, args, *reply.
		if mtype.NumIn() != 3 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d input parameters; needs exactly three\n", mname, mtype.NumIn())
			}
			continue
		}
		// 第一个参数不必是指针类型
		argType := mtype.In(1)
		if !isExportedOrBuiltinType(argType) {
			if reportErr {
				log.Printf("rpc.Register: argument type of method %q is not exported: %q\n", mname, argType)
			}
			continue
		}
		// 第二个参数必须是指针类型
		replyType := mtype.In(2)
		if replyType.Kind() != reflect.Ptr {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not a pointer: %q\n", mname, replyType)
			}
			continue
		}
		if !isExportedOrBuiltinType(replyType) {
			if reportErr {
				log.Printf("rpc.Register: reply type of method %q is not exported: %q\n", mname, replyType)
			}
			continue
		}
		// Method 只需要一个类型为 typeOfError 的返回值
		if mtype.NumOut() != 1 {
			if reportErr {
				log.Printf("rpc.Register: method %q has %d output parameters; needs exactly one\n", mname, mtype.NumOut())
			}
			continue
		}
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

// A value sent as a placeholder for the server's response value when the server
// receives an invalid request. It is never decoded by the client since the Response
// contains an error when it is used.
// 无效请求的默认响应
var invalidRequest = struct{}{}

func (server *Server) sendResponse(sending *sync.Mutex, req *Request, reply interface{}, codec ServerCodec, errmsg string) {...}

// 调用次数
func (m *methodType) NumCalls() (n uint) {...}

// 客户端请求后，服务端通过 call 调用相应 服务
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
	// The return value for the method is an error.
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

func (c *gobServerCodec) Close() error {
	if c.closed {
		// Only call c.rwc.Close once; otherwise the semantics are undefined.
		return nil
	}
	c.closed = true
	return c.rwc.Close()
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

// ServeRequest 与 ServeCodec 类似但只会同步地服务单个 request
// 它不会关闭 codec，即使在结束之后
func (server *Server) ServeRequest(codec ServerCodec) error {...}

// 获取头结点的 request
func (server *Server) getRequest() *Request {
	server.reqLock.Lock()
	req := server.freeReq
	if req == nil {
		req = new(Request)
	} else {
		server.freeReq = req.next
		*req = Request{}
	}
	server.reqLock.Unlock()
	return req
}

// 将当前 req 插入到 freeReq 链表头部
func (server *Server) freeRequest(req *Request) {
	server.reqLock.Lock()
	req.next = server.freeReq
	server.freeReq = req
	server.reqLock.Unlock()
}

// 与 request 同理
func (server *Server) getResponse() *Response {...}

func (server *Server) freeResponse(resp *Response) {...}

func (server *Server) readRequest(codec ServerCodec) (service *service, mtype *methodType, req *Request, argv, replyv reflect.Value, keepReading bool, err error) {...}

func (server *Server) readRequestHeader(codec ServerCodec) (svc *service, mtype *methodType, req *Request, keepReading bool, err error) {...}

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

// Register publishes the receiver's methods in the DefaultServer.
func Register(rcvr interface{}) error { return DefaultServer.Register(rcvr) }

func RegisterName(name string, rcvr interface{}) error { return DefaultServer.RegisterName(name, rcvr) }

// ServerCodec 接口实现了 RPC 会话的服务端一侧 RPC 请求的读取和 RPC 回复的写入。
// 服务端通过成对调用方法 ReadRequestHeader 和 ReadRequestBody 从连接读取请求，
// 然后调用WriteResponse来写入回复。服务端在结束该连接的事务时调用Close方法。
// ReadRequestBody 可以使用nil参数调用，以强制请求的主体被读取然后丢弃
type ServerCodec interface {
	ReadRequestHeader(*Request) error
	ReadRequestBody(interface{}) error
	// WriteResponse must be safe for concurrent use by multiple goroutines.
	WriteResponse(*Response, interface{}) error

	Close() error
}

func ServeConn(conn io.ReadWriteCloser) { DefaultServer.ServeConn(conn) }

func ServeCodec(codec ServerCodec) { DefaultServer.ServeCodec(codec) }

func ServeRequest(codec ServerCodec) error { return DefaultServer.ServeRequest(codec) }

func Accept(lis net.Listener) { DefaultServer.Accept(lis) }

// Can connect to RPC service using HTTP CONNECT to rpcPath.
var connected = "200 Connected to Go RPC"

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

func HandleHTTP() { DefaultServer.HandleHTTP(DefaultRPCPath, DefaultDebugPath) }
```

### 2. rpc/client.go

![image](https://raw.githubusercontent.com/fengyfei/wizard/d16dc5a4e3a33bdfd2b37175355a990d83a6f873/go/rpc/images/rpc_client.png?token=AXUDI0fDqAwTpP2P5Wc9JNY0bMSPMuzdks5ascxswA%3D%3D)

```go
// ServerError 代表远程 RPC 连接返回的错误
type ServerError string

func (e ServerError) Error() string {
	return string(e)
}

var ErrShutdown = errors.New("connection is shut down")

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

// 注册传入的 call，然后编码并发送请求
func (client *Client) send(call *Call) {...}

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

func (call *Call) done() {
	select {
	case call.Done <- call:
		// ok
	default:
		// We don't want to block here. It is the caller's responsibility to make
		// sure the channel has enough buffer space. See comment in Go().
		if debugLog {
			log.Println("rpc: discarding Call reply due to insufficient Done chan capacity")
		}
	}
}

// NewClient 返回一个新的 Client，以管理对连接另一端的服务的请求。
// 它添加缓冲到连接的写入侧，以便将回复的头域和有效负载作为一个单元发送。
func NewClient(conn io.ReadWriteCloser) *Client {
	encBuf := bufio.NewWriter(conn)
	client := &gobClientCodec{conn, gob.NewDecoder(conn), gob.NewEncoder(encBuf), encBuf}
	return NewClientWithCodec(client)
}

// NewClientWithCodec 与 NewClient 类似但使用指定的编解码器
func NewClientWithCodec(codec ClientCodec) *Client {
	client := &Client{
		codec:   codec,
		pending: make(map[uint64]*Call),
	}
	go client.input()
	return client
}

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

// Close 调用编解码器含的 Close 函数. 如果连接已关闭则会返回 ErrShutdown
func (client *Client) Close() error {
	client.mutex.Lock()
	if client.closing {
		client.mutex.Unlock()
		return ErrShutdown
	}
	client.closing = true
	client.mutex.Unlock()
	return client.codec.Close()
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

参考

- Go RPC 开发指南 https://www.gitbook.com/book/smallnest/go-rpc-programming-guide/details

- Go官方库RPC开发指南  http://colobu.com/2016/09/18/go-net-rpc-guide/

- build-web-application-with-golang https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/08.4.md

- wikipedia https://en.wikipedia.org/wiki/Remote_procedure_call

- How RPC Works https://technet.microsoft.com/en-us/library/cc738291%28v=ws.10%29.aspx?f=255&MSPPError=-2147217396