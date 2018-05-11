# mcache.go

##结构

###mcache

```go
type mcache struct {
   
   next_sample int32   
   local_scan  uintptr 
  
    // 小对象分配器，小于 16 byte 的小对象都会通过 tiny 来分配
   tiny             uintptr
   tinyoffset       uintptr
   local_tinyallocs uintptr 
    
   alloc [numSpanClasses]*mspan 
    
   stackcache [_NumStackOrders]stackfreelist

    // GC相关，在 GC 过程中会进行刷新。
   local_nlookup    uintptr  // 指针查找次数               
   local_largefree  uintptr  //                
   local_nlargefree uintptr                 
   local_nsmallfree [_NumSizeClasses]uintptr 
}
```

### gclink

```go
// block 链表的节点。代码应该将 gclink 的引用存储为 gclinkptr，而不是 *gclink
type gclink struct {
   next gclinkptr
}

type gclinkptr uintptr

// 返回 p 的 *gclink 形式，用于访问。 
func (p gclinkptr) ptr() *gclink {
   return (*gclink)(unsafe.Pointer(p))
}
```

### statckfreelist

```go
// 空 stack 链表
type stackfreelist struct {
   list gclinkptr  
   size uintptr   
}

// dummy MSpan that contains no free objects.
var emptymspan mspan
```



## 函数

###mcache 相关

```go
// 分配mcache内存
func allocmcache() *mcache {
   lock(&mheap_.lock)
    
    // 调用 heap 的 fixalloc的函数，初始化一个 mcache
    // c 是一个 mcache 类型的指针
   c := (*mcache)(mheap_.cachealloc.alloc())
   unlock(&mheap_.lock)
   for i := range c.alloc {
      c.alloc[i] = &emptymspan
   }
   c.next_sample = nextSample()
   return c
}

// 释放 mcache 
func freemcache(c *mcache) {
    // 启动 stw,调用 releaseAll 函数进行释放内存的第一步
   systemstack(func() {
      c.releaseAll()
      stackcache_clear(c)

   // 将剩下的 mcache（基本是个空壳）归还给 cachealloc(就是放回 cachealloc 中的 free list)。
      lock(&mheap_.lock)
      purgecachedstats(c)
      mheap_.cachealloc.free(unsafe.Pointer(c))
      unlock(&mheap_.lock)
   })
}

// 填充 mcache
// 如果 mcache 中的 某个规格的 mspan 内存不够，调用 refill 函数从对应的 mcentral 中获取空 mspan
func (c *mcache) refill(spc spanClass) *mspan {
   _g_ := getg()

   _g_.m.locks++
   
    //
   s := c.alloc[spc]

   if uintptr(s.allocCount) != s.nelems {
      throw("refill of span with free space remaining")
   }

   if s != &emptymspan {
      s.incache = false
   }

    // 从 mcentral 中获得一个有空闲内存的 span
   s = mheap_.central[spc].mcentral.cacheSpan()
   if s == nil {
      throw("out of memory")
   }

   if uintptr(s.allocCount) == s.nelems {
      throw("span has no free space")
   }

   c.alloc[spc] = s
   _g_.m.locks--
   return s
}

// 释放 mcache 中的内存
// 把 alloc 中未用完的内存归还给 mcentral
func (c *mcache) releaseAll() {
   for i := range c.alloc {
      s := c.alloc[i]
      if s != &emptymspan {
         mheap_.central[i].mcentral.uncacheSpan(s)
         c.alloc[i] = &emptymspan
      }
   }
  
    // 将小对象分配器清零
   c.tiny = 0
   c.tinyoffset = 0
}
```

