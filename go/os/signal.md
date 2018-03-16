# Signal

信号(Signal)是 Linux, 类 Unix 和其它 POSIX 兼容的操作系统中用来进程间通讯的一种方式。

一个信号就是一个异步的通知，发送给某个进程，或者同进程的某个线程，告诉它们某个事件发生了。

当信号发送到某个进程中时，操作系统会中断该进程的正常流程，并进入相应的信号处理函数执行操作，完成后再回到中断的地方继续执行。

如果目标进程先前注册了某个信号的处理程序(signal handler),则此处理程序会被调用，否则缺省的处理程序被调用。

Linux/macOS 可通过下面的命令，给特定进程发送信号量。

```sh
kill -1/2/3/6/9 PID 
```

## 源码分析

- 相关结构定义：

```go
// 全局信号量处理
var handlers struct {
	sync.Mutex
	m   map[chan<- os.Signal]*handler
	ref [numSig]int64                   // 信号量引用计数(有多少个 handler 处理该信号量)
}

// 学习位掩码使用方式
type handler struct {
	// 位掩码长度
	mask [(numSig + 31) / 32]uint32	// (numSig + 31) / 32 是为了获取足够的位掩码空间
}
```

- 位掩码操作

```go
// sig/32 --> 索引
// >> uint(sig&31) --> 位
// &1 : 验证是否置位
func (h *handler) want(sig int) bool {
	return (h.mask[sig/32]>>uint(sig&31))&1 != 0
}

// 设置信号量对应的位掩码
func (h *handler) set(sig int) {
	h.mask[sig/32] |= 1 << uint(sig&31)
}

// 清除信号量对应的位掩码
func (h *handler) clear(sig int) {
	h.mask[sig/32] &^= 1 << uint(sig&31)
}
```

- Nofity 操作

```go
func Notify(c chan<- os.Signal, sig ...os.Signal) {
	// 参数检查
	if c == nil {
		panic("os/signal: Notify using nil channel")
	}

	handlers.Lock()
	defer handlers.Unlock()

	h := handlers.m[c]
	
	// 映射表没有该项，创建
	if h == nil {
		if handlers.m == nil {
			handlers.m = make(map[chan<- os.Signal]*handler)
		}
		h = new(handler)
		handlers.m[c] = h
	}

	add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			
			// 首次引用，开启信号量
			if handlers.ref[n] == 0 {
				enableSignal(n)
			}
			handlers.ref[n]++
		}
	}

	// 已有，修改
	if len(sig) == 0 {
		for n := 0; n < numSig; n++ {
			add(n)
		}
	} else {
		for _, s := range sig {
			add(signum(s))
		}
	}
}
```

- Stop

```go
func Stop(c chan<- os.Signal) {
	handlers.Lock()
	defer handlers.Unlock()

	// 从全局映射中清除
	h := handlers.m[c]
	if h == nil {
		return
	}
	delete(handlers.m, c)

	// 修改信号量引用计数
	for n := 0; n < numSig; n++ {
		if h.want(n) {
			handlers.ref[n]--
			
			// 最后一个关注，禁用该信号量
			if handlers.ref[n] == 0 {
				disableSignal(n)
			}
		}
	}
}
```

- cancel

```go
func cancel(sigs []os.Signal, action func(int)) {
	handlers.Lock()                                 // 锁保护
	defer handlers.Unlock()

	remove := func(n int) {
		var zerohandler handler

		for c, h := range handlers.m {
			if h.want(n) {                          // 如果处理信号量
				handlers.ref[n]--                   // 引用计数减一
				h.clear(n)                          // 本 handler 不再处理
				if h.mask == zerohandler.mask {     // 已经没有关注的信号量
					delete(handlers.m, c)           // 从全局移除
				}
			}
		}

		action(n)                                   // 处理信号量
	}

	if len(sigs) == 0 {                             // 处理全部
		for n := 0; n < numSig; n++ {
			remove(n)
		}
	} else {
		for _, s := range sigs {                    // 处理传入的信号量
			remove(signum(s))
		}
	}
}
```

## References

- [Linux Signal Manual](http://man7.org/linux/man-pages/man7/signal.7.html)
