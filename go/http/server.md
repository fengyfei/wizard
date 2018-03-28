### server

![image](https://github.com/fengyfei/wizard/raw/master/go/http/images/server.png)

server 与 conn 等接口
```go
type Server struct {
    Addr              string        // 要监听的 TCP 地址
    Handler           Handler       // 调用的 handler, 如果为空则用 http.DefaultServeMux
    TLSConfig         *tls.Config   // 用于 ServeTLS 和 ListenAndServeTLS
    ReadTimeout       time.Duration // 读取完整 request (包括 body) 的最大时长，可以和 ReadHeaderTimeout 同时使用
    ReadHeaderTimeout time.Duration // 读取 request headers 的最大时长
    WriteTimeout      time.Duration // 写 response 的最大时长
    IdleTimeout       time.Duration // 当 keepalive 开启时等待下个 request 的最大时长，此值为空时使用 ReadTimeout 值代替，ReadTimeout 也为空使用 ReadHeaderTimeout 代替
    MaxHeaderBytes    int           // 解析 request headers 里键值对的最大字节数(包含请求行)，不限制 body. 如果为 0, 使用 DefaultMaxHeaderBytes 代替
    TLSNextProto      map[string]func(*Server, *tls.Conn, Handler) // 当'应用层协议协商(NPN/ALPN)'时发生协议升级时，TLSNextProto 需要指定可选的 function 去接管 TLS 连接
    ConnState         func(net.Conn, ConnState) // 指定一个可选的钩子函数，由 client 连接状态改变触发
    ErrorLog          *log.Logger   // 指定一个可选的 logger 接收错误日志. 如果为空则由 log 包接管
    disableKeepAlives int32         // 在 SetKeepAlivesEnabled 中设置，为 1 表示取消长连接，为 0 保持长连接(默认)
    inShutdown        int32         // 非零代表 in Shutdown
    nextProtoOnce     sync.Once     // 设置 HTTP/2
    nextProtoErr      error         // http2.ConfigureServer 的结果
    mu                sync.Mutex
    listeners         map[net.Listener]struct{}
    activeConn        map[*conn]struct{}
    doneChan          chan struct{} // doneChan 代表任务结束
    onShutdown        []func()      // 通过 RegisterOnShutdown 注册，在 Shutdown 时调用当中的钩子函数
}

// 此接口由 ResponseWriters 执行去检测连接是否已断开，此机制允许客户端断开后服务端取消一个长连接
type CloseNotifier interface {
    CloseNotify() <-chan bool
}

// conn 代表服务端的 HTTP 连接
type conn struct {
    server     *Server
    cancelCtx  context.CancelFunc   // 撤销连接层的 context，读写出错时会调用
    rwc        net.Conn             // 
    remoteAddr string               // rwc.RemoteAddr().String()
    tlsState   *tls.ConnectionState // TLS 连接状态，nil 代表非 TSL
    werr       error                // rwc 写入时的首个错误(bufw 写入时)
    r          *connReader          // 一个 *conn 使用的 io.reader 封装，存有 bufr 的读取内容
    bufr       *bufio.Reader        // 从 r 读取
    bufw       *bufio.Writer        // 要写入 checkConnErrorWriter{c} 的缓冲
    lastMethod string
    curReq     atomic.Value // 存入 *response (response 中包含 request)
    curState   atomic.Value // 存入 ConnState
    mu         sync.Mutex   // 保护 hijackedv
    hijackedv  bool         // 代表连接是否已经被 hijacke
}

// 一个 ctx 带有一个截止期限，一个取消信号，或者其他绑定值
// 其函数可以被多个 goroutines 同时使用
// 一个请求过来时可能会涉及到多个 goroutines，Ctx 可以控制关闭与之相关联和派生出的子 ctx 相关联的 goroutines
type Context interface {
	// Deadline 方法是获取设置的截止时间，第一个返回值是截止时间，到了这个时间点，Context会自动发起取消请求；
	// 第二个返回值 ok==false 时表示没有设置截止时间，如果需要取消的话，需要调用 cancel 函数进行取消，取消操作包括派生出去的子 Ctx
	Deadline() (deadline time.Time, ok bool)
	// 在 goroutine 中，如果该方法返回的 chan 可以读取，则意味着 parent context 已经发起了取消请求，
	// 我们通过 Done 方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源
    Done() <-chan struct{}
    // 如果 Done 还没关闭，Err 返回 nil
    // 如果 Done 以及关闭，返回非空 err，告知 Ctx 因何取消
    Err() error
    Value(key interface{}) interface{} // 键值对形式，与 Ctx 绑定，可以为空
}
```

监听函数
```go
// Serve 接收 listener 上过来的连接，并为每个连接创建 service 线程
// 在 service 线程中会读取 request 并调用 srv.Handler 进行服务
// handler 参数一般传 nil 就行，代表使用的是 DefaultServeMux
func Serve(l net.Listener, handler Handler) error { // HTTPS: ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error
	srv := &Server{Handler: handler}
	return srv.Serve(l) // HTTPS: srv.ServeTLS(l, certFile, keyFile)
}

// func HelloServer(w http.ResponseWriter, req *http.Request) {
//     io.WriteString(w, "hello, world!\n")
// }
//
// func main() {
//     http.HandleFunc("/hello", HelloServer)
//     log.Fatal(http.ListenAndServe(":12345", nil))
// }
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

// err := http.ListenAndServeTLS(":10443", "cert.pem", "key.pem", nil)
// HTTPS 方式，可以使用 crypto/tls 中的 generate_cert.go 生成 cert.pem 和 key.pem
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServeTLS(certFile, keyFile)
}

// ListenAndServe 监听 srv.Addr 地址上的 tcp 网络，然后调用 Serve 服务连接，连接会设置 keep-alives
// 如果 srv.Addr 为空则用 ":http" 代替
// ListenAndServe 总是返回非空 err
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	// HTTP: 
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
	
	// HTTPS 方式调用 ListenAndServeTLS(certFile, keyFile string) error
	// 与 ListenAndServe 类似，只是最后要关闭 ln 并返回 srv.ServeTLS
	// defer ln.Close() 
	// return srv.ServeTLS(tcpKeepAliveListener{ln.(*net.TCPListener)}, certFile, keyFile)
}
```

server 的服务函数
```go
func (srv *Server) ServeTLS(l net.Listener, certFile, keyFile string) error {
	// 在 srv.Serve 之前尝试设置 HTTP/2
	// setupHTTP2_ServeTLS 中调用 onceSetNextProtoDefaults_Serve，只有 srv.TLSNextProto 为 nil 时才可以设置 HTTP/2
	if err := srv.setupHTTP2_ServeTLS(); err != nil {
		return err
	}

	config := cloneTLSConfig(srv.TLSConfig)
	if !strSliceContains(config.NextProtos, "http/1.1") { // strSliceContains 判断是否包含字符串
		config.NextProtos = append(config.NextProtos, "http/1.1")
	}

	configHasCert := len(config.Certificates) > 0 || config.GetCertificate != nil
	if !configHasCert || certFile != "" || keyFile != "" {
		var err error
		config.Certificates = make([]tls.Certificate, 1)
		config.Certificates[0], err = tls.LoadX509KeyPair(certFile, keyFile) // LoadX509KeyPair 解析证书，文件中必须含有 PEM 编码数据
		// PEM (Privacy Enhancement Message)，定义见 RFC1421，是一种基于 base64 的编码格式
		if err != nil {
			return err
		}
	}

	tlsListener := tls.NewListener(l, config)
	return srv.Serve(tlsListener)
}

// 若启用 HTTP/2，在调用 Serve 前需要根据 listener's TLS Config 初始化 srv.TLSConfig
// Serve 总是返回非空的 err，在 Shutdown 或 Close 后返回 ErrServerClosed
// Close 是立即关闭 Server 和与之相关的 listeners 和 connections，而 shutdown 是逐步关闭 listeners 和闲置的 connections，两者不会管已被 hijack 的连接
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	if fn := testHookServerServe; fn != nil { // 如果钩子函数 testHookServerServe 非空则调用
		fn(srv, l)
	}
	var tempDelay time.Duration // accept 失败时 sleep 多长时间

    // setupHTTP2_Serve 和 setupHTTP2_ServeTLS 两者都是调用 onceSetNextProtoDefaults() 去尝试设置 HTTP/2
    // 只是考虑到多并发情况下的 Serve 请求，setupHTTP2_Serve 采用了更保守的政策去设置 HTTP/2
    // setupHTTP2_Serve 先调用 shouldConfigureHTTP2ForServe 判断是否应该为 Server.Serve 设置 HTTP/2
    // shouldConfigureHTTP2ForServe 中如果 srv.TLSConfig 为 nil 或者 srv.TLSConfig.NextProtos 包含 "h2" 字样返回真，否则返回假，
	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	srv.trackListener(l, true) // 将 l 添加进 server.listeners
	defer srv.trackListener(l, false) // 结束后删去 l

	baseCtx := context.Background() // baseContext 会一直存在，但没有值也没有 deadline，用于主函数或者初始化或者测试或者顶层接收请求的 context
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	// WithValue 返回 baseCtx 的副本，副本内的值是一个键值对 ServerContextKey - srv
	// ServerContextKey = &contextKey{"http-server"} 与其绑定的 value 类型为 *Server
	for {
		rw, e := l.Accept() // 接收到连接
		if e != nil {
			select {
			case <-srv.getDoneChan(): // server 已关闭
				return ErrServerClosed
			default:
			}
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		// conn.setState 根据传入的状态调用 trackConn 来设置 server.activeConn 集合，再改变当前 conn.curState
		// 如果 server 设置了 ConnState 这个钩子函数，就调用
		c.setState(c.rwc, StateNew)
		go c.serve(ctx)
	}
}
```

server.Serve 最后调用 conn.serve，在此函数中调用 `serverHandler{c.server}.ServeHTTP(w, w.req)` 转入路由模块
```go
func (c *conn) serve(ctx context.Context) {
	c.remoteAddr = c.rwc.RemoteAddr().String()
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
	// LocalAddrContextKey = &contextKey{"local-addr"} 与其绑定的 value 类型是 net.Addr
	defer func() {
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10 // 64 KB
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() { // 已经被 hijack 的连接不用管理，由 hijack 的调用者处理
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()

	if tlsConn, ok := c.rwc.(*tls.Conn); ok { // HTTPS
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
		if err := tlsConn.Handshake(); err != nil {
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState() // 获取当前 TLS 连接的详细信息
		// NegotiatedProtocol 协商的协议，validNPN 判断 proto 是否属于 "", "http/1.1", "http/1.0" 之一，不属于返回真
		if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initNPNRequest{tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h) // 发生协议切换时触发钩子函数
			}
			return
		}
	}

	// HTTP/1.x flowing

	ctx, cancelCtx := context.WithCancel(ctx)
	// WithCancel 返回 &c, func() { c.cancel(true, Canceled) }
	// ctx.cancel close ctx.done 取消所有 ctx 的 children，如果第一个参数为 true，则把 ctx 从其 parent 的 children 列表删去
	c.cancelCtx = cancelCtx
	defer cancelCtx() // 关闭 ctx，以及相关 goroutines

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		w, err := c.readRequest(ctx) // 读取 request 返回 response 和可能的 err
		if c.r.remain != c.server.initialReadLimitSize() { // remain 代表 io.reader 剩余空间，initialReadLimitSize 返回 int64(srv.MaxHeaderBytes > 0 ? srv.MaxHeaderBytes : DefaultMaxHeaderBytes) + 4096
			c.setState(c.rwc, StateActive) // StateActive 代表连接已经从 request 读到数据
		}
		if err != nil {
			const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

			if err == errTooLarge { // errors.New("http: request too large")
				const publicErr = "431 Request Header Fields Too Large"
				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				c.closeWriteAndWait()
				// closewrite flush 所有缓存的数据并发送一个 FIN 包（如果客户端是通过 TCP 连接的），表示我们这边已结束，然后 sleep 500 ms
				return
			}
			if isCommonNetReadError(err) {
				// err 是否是 io.EOF 或者是网络超时 (net.Error) 或者是读 request 的 net.OpError 之一
				return
			}

			publicErr := "400 Bad Request"
			if v, ok := err.(badRequestError); ok {
				publicErr = publicErr + ": " + string(v)
			}

			fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
			return
		}

		// request Header : Expect 100 Continue
		req := w.req
		if req.expectsContinue() {
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// after first '100 Continue' request, wrapper response with 'HTTP/1.1 100 Continue'
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed() // response with status code 417 (Expectation Failed)
			return
		}

		c.curReq.Store(w)

		if requestBodyRemains(req.Body) { // 之后是否还能从 body 读取到数据，true 表示能继续读(未到 io.EOF)
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead) // 当 body 读到 EOF，调用传入的 startBackgroundRead 函数
		} else { // 长连接下 HTTP 管线化请求时的处理
			if w.conn.bufr.Buffered() > 0 {
				// [HTTP pipelining](https://zh.wikipedia.org/wiki/HTTP%E7%AE%A1%E7%B7%9A%E5%8C%96)
				w.conn.r.closeNotifyFromPipelinedRequest() // closeNotify()
			}
			w.conn.r.startBackgroundRead()
		}

		serverHandler{c.server}.ServeHTTP(w, w.req) // server.Handler == nil -> DefaultServeMux.ServeHTTP
		w.cancelCtx()
		if c.hijacked() {
			return
		}
		w.finishRequest()
		if !w.shouldReuseConnection() { // tcp 连接是否可以继续使用
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				// requestBodyLimitHit 在 requestTooLarge 函数中设置，当此值为真，停止读取后续的 request 和输入
				// closedRequestBodyEarly 表示连接之前是否已关闭
				c.closeWriteAndWait()
			}
			return
		}
		c.setState(c.rwc, StateIdle) // StateIdle 表示此连接已处理完一个 request 并处于 keep-alive 状态，等待后续 request
		c.curReq.Store((*response)(nil))

		if !w.conn.server.doKeepAlives() { // doKeepAlives 判断是否满足 disableKeepAlives == 0 && inShutdown == 0 (处于 keep-alive 模式且不在 shutdown 状态)
			// We're in shutdown mode. We might've replied
			// to the user without "Connection: close" and
			// they might think they can send another
			// request, but such is life with HTTP/1.1.
			return
		}

		if d := c.server.idleTimeout(); d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
		// SetReadDeadline 设置后续读去调用的截止时间，如果传入零值表示不会 timeout
		c.rwc.SetReadDeadline(time.Time{})
	}
}
```

---

流程：
当一个请求 request 进来的时候，server 会依次根据 ServeMux.m 中的 string（路由表达式）来一个一个匹配，
如果找到了可以匹配的 muxEntry，就取出 muxEntry.h，这是个 handler，
调用 handler 中的 ServeHTTP（ResponseWriter, *Request）来组装 Response，并返回。

---

路由接口
```go
// ResponseWriter 接口用于 HTTP handler 生成 response
// 在 Handler.ServeHTTP 返回后，ResponseWriter 不应该再被使用
type ResponseWriter interface {
    Header() Header             // Header() 返回 WriteHeader 要发送的 Header map 集合
    Write([]byte) (int, error)  // Write 写入响应的 body
    WriteHeader(statusCode int) // 这个方法发送 Response 的 Header 和传入的 HTTP 状态码
}

// Flusher 由 ResponseWriters 执行去允许 HTTP handler 将缓存中的数据推给客户端, 默认的 HTTP/1.x 和 HTTP/2 ResponseWriter 支持 Flusher，
// 但是 ResponseWriter 的封装可能会不支持，Handlers 在运行时需要测试是否支持此函数
// 即使 ResponseWriters 支持 Flush，如果客户端使用了 HTTP proxy，直到响应结束，缓存的数据也有可能到达不了客户端
type Flusher interface {
	Flush()
}

// Hijacker 接口由 ResponseWriters 执行去允许 HTTP handler 接管连接
// 默认的 ResponseWriter 支持 HTTP/1.x 连接下的 Hijacker，但是 HTTP/2 连接不支持，HTTP/2 多路复用等情况不适合使用 Hijack 。
// ResponseWriter 封装也可能不支持 Hijacker. Handlers 在运行时需要测试是否支持此函数
type Hijacker interface {
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}

// ServeMux 类型是 HTTP 请求的路由规则转换器。它会将每一个接收的请求的 URL 与一个注册路由的列表进行匹配，并调用和 URL 最匹配的 handler.
// 匹配到多个时较长的模式优先于较短的模式，模式也可以主机名开始，表示只匹配该主机上的路径，指定主机的模式优先于一般的模式，
// ServeMux 还会规范化请求的 URL 路径，将任何包含"."或".."元素的请求重定向到等价的没有这两种元素的URL
type ServeMux struct {
	mu    sync.RWMutex // 读写锁
	m     map[string]muxEntry // 路由规则，一个 string 对应一个 mux 实体，这里的 string 就是注册的路由表达式
	hosts bool // 是否在任意的规则中带有 host 信息
}

type muxEntry struct {
    h        Handler // 这个路由表达式对应哪个 handler
    pattern  string  // 固定的、由根开始的路径，如 "/favicon.ico"，或由根开始的子树，如 "/images/"，也可以主机名开头
}

// 一个 Handler 响应一个 HTTP 请求
// ServeHTTP 应该将回复的头域和数据写入 ResponseWriter 接口然后返回。返回标志着该请求已经结束，HTTP服务端可以转移向该连接上的下一个请求。
// 在 ServeHTTP 调用结束之后或者并发执行时，使用 ResponseWriter 或者读取请求体是不可取的
// handler 应该第一时间读取请求体并作出应答，在向 ResponseWriter 写入数据后就不能读取 request body 了. 同时 handler 不应该修改传入的 request
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// HandlerFunc(f) 是一个调用 f 的 handler
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
**请求 - 响应实例**

这里实现了一个 `404 not found` 响应
```go
func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) } // 定义 handler

func NotFoundHandler() Handler { return HandlerFunc(NotFound) }
```

server 导出的注册函数使用 DefaultServeMux 相应方法
```go
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	mux.m[pattern] = muxEntry{h: handler, pattern: pattern} // 注册成功

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

ServeHTTP 调用 Handler() 给 request 分派与 request URL 最匹配的 handler

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) { // ProtoAtLeast 判断是否大于等于协议最低标准，第一个参数是 major 版本号，第二个参数是 minor 版本号，即 http/1.1
			w.Header().Set("Connection", "close") // 小于要求则在响应头返回关闭信息
		}
		w.WriteHeader(StatusBadRequest) // 状态码 400
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r) // 调用对应 handler 的 ServeHTTP，即执行注册好的 handler 函数，比如 NotFound 函数
}
```

```go
// Handler 通过判断 r.Method, r.Host, and r.URL.Path 返回与 request 对应的 handler
// 此函数总会返回非空的 handler. 如果 path 不符合规范形式，返回的是内部生成的重定向到规范路径的 handler
// 如果 host 包含端口，匹配 handlers 时会忽略端口。第二个参数返回已注册的与请求匹配的路由
// 如果没有已注册的 handler 与请求匹配, 则返回 ``page not found'' handler 和空的 pattern
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	if r.Method == "CONNECT" {
		// redirectToPathSlash 判断 path 是否需要追加 "/"，因为存在 "path + /"已注册但 "path"
		// 本身未注册的情况。如果需要追加 "/"，则返回追加的 url 和 true
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}

		return mux.handler(r.Host, r.URL.Path)
	}

	host := stripHostPort(r.Host) // 去掉 ":<port>"
	path := cleanPath(r.URL.Path) // 规范 path 格式，比如缺失多余'/'、存在相对路径'.'、'..'等

	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
	}

    // 修改 request 的不规范路径 
	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		url := *r.URL
		url.Path = path
		return RedirectHandler(url.String(), StatusMovedPermanently), pattern
	}

	return mux.handler(host, r.URL.Path)
}

// 在 ServerMux.handler 中当匹配不到注册的路由时返回 NotFoundHandler
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	if mux.hosts {
		h, pattern = mux.match(host + path) // match 根据完整 URL 优先匹配 handler
	}
	if h == nil {
		h, pattern = mux.match(path) // 如果 URL 匹配不到再根据路径匹配
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}
```

### References

- [Package http](https://golang.org/pkg/net/http/)
- [Go Web 编程](https://astaxie.gitbooks.io/build-web-application-with-golang/zh/)
- [wiki NPN/ALPN](https://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E5%B1%82%E5%8D%8F%E8%AE%AE%E5%8D%8F%E5%95%86)
- [wiki TLS](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A)
