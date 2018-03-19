### net/http/server.go

```
// ServeMux 类型是 HTTP 请求的路由规则转换器。它会将每一个接收的请求的 URL 与一个注
// 册模式的列表进行匹配，并调用和 URL 最匹配的模式的 handler.
// 模式是固定的、由根开始的路径，如"/favicon.ico"，或由根开始的子树，
// 如"/images/"（注意结尾的斜杠）。较长的模式优先于较短的模式，
// 因此如果模式"/images/"和"/images/thumbnails/"都注册了处理器，
// 后一个处理器会用于路径以"/images/thumbnails/"开始的请求，
// 前一个处理器会接收到其余的路径在"/images/"子树下的请求。
// 注意，因为以斜杠结尾的模式代表一个由根开始的子树，模式"/"会匹配所有的未被其他注册的模式匹配的路径，而不仅仅是路径"/"。
// 模式也能（可选地）以主机名开始，表示只匹配该主机上的路径。指定主机的模式优先于一般的模式，
// 因此一个注册了两个模式"/codesearch"和"codesearch.google.com/"的处理器不会接管目标为"http://www.google.com/"的请求。
// ServeMux还会注意到请求的 URL 路径的无害化，将任何路径中包含"."或".."元素的请求重定向到等价的没有这两种元素的URL。（参见path.Clean函数）
type ServeMux struct {
	mu    sync.RWMutex // 读写锁
	m     map[string]muxEntry // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
	hosts bool // 是否在任意的规则中带有host信息
}

type muxEntry struct {
    h        Handler // 这个路由表达式对应哪个handler
    pattern  string  // 匹配字符串
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
**使用实例**

这里实现了一个 `404 not found` 响应
```
func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) }

// NotFoundHandler 返回一个 '404 page not found' 响应的 handler
func NotFoundHandler() Handler { return HandlerFunc(NotFound) } // 函数的类型转换
```

在 ServerMux.handler 中当匹配不到注册的路由时返回 NotFoundHandler
```
// handler is the main implementation of Handler.
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}

// Handler 通过判断 r.Method, r.Host, and r.URL.Path 返回与 request 对应的 handler
// 此函数总会返回非空的 handler. 如果 path 不符合规范形式，返回的是内部生成的重定向到规范路径的 handler
// 如果 host 包含端口，匹配 handlers 时会忽略端口。
// The path and host are used unchanged for CONNECT requests.
// 第二个参数返回已注册的与请求匹配的路由
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
```
ServeHTTP 调用 Handler() 给 request 分派与 request URL 最匹配的 handler

```
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r) // 调用对应 handler 的 ServeHTTP
}
```

```
// 注册 handler 方法
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
	mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}

// 注册 handler 方法（直接使用func注册）
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```
server 导出的注册函数使用 DefaultServeMux 相应方法
```
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```
---

正常流程：
当一个请求 request 进来的时候，server 会依次根据 ServeMux.m 中的 string（路由表达式）来一个一个匹配，如果找到了可以匹配的 muxEntry，就取出 muxEntry.h，这是个 handler，调用 handler 中的 ServeHTTP（ResponseWriter, *Request）来组装 Response，并返回。

---

其他接口，未完待续
```
// ResponseWriter 接口用于 HTTP handler 生成 response
// 在 Handler.ServeHTTP 返回后，ResponseWriter 不应该再被使用
type ResponseWriter interface {
    // Header() 返回 WriteHeader 要发送的 Header map 集合
	Header() Header
	// Write 写入响应的 body
	Write([]byte) (int, error)
	// 这个方法发送 Response 的 Header 和传入的 HTTP 状态码
	WriteHeader(statusCode int)
}

// Flusher 由 ResponseWriters 执行去允许 HTTP handler 将缓存中的数据推给客户端
// 默认的 HTTP/1.x 和 HTTP/2 ResponseWriter 支持 Flusher，
// 但是 ResponseWriter 的封装可能会不支持，Handlers 在运行时需要测试是否支持此函数
// 即使 ResponseWriters 支持 Flush，如果客户端使用了 HTTP proxy，直到响应结束，
// 缓存的数据也可能到达不了客户端
type Flusher interface {
	Flush()
}

// Hijacker 接口由 ResponseWriters 执行去允许 HTTP handler 接管连接
// 默认的 ResponseWriter 支持 HTTP/1.x 连接下的 Hijacker，但是 HTTP/2 连接不支持。
// ResponseWriter 封装也可能不支持 Hijacker. Handlers 在运行时需要测试是否支持此函数
type Hijacker interface {
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}

// conn 代表服务端的 HTTP 连接
type conn struct {
	server *Server
	cancelCtx context.CancelFunc
	rwc net.Conn
	remoteAddr string
	tlsState *tls.ConnectionState
	werr error
	r *connReader
	bufr *bufio.Reader
	bufw *bufio.Writer
	lastMethod string
	curReq atomic.Value // of *response (which has a Request in it)
	curState atomic.Value // of ConnState
	mu sync.Mutex
	hijackedv bool
}
```

### References

- [Package http](https://golang.org/pkg/net/http/)
- [Golang Http Server源码阅读](https://studygolang.com/articles/1740)
