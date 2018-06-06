# Connection Management

## 监听

首先，检查是否有配置监听的 IP 及 端口，如果没有，监听全部网络接口

```C
// 将首个绑定地址设置为 NULL，会监听全部端口
if (server.bindaddr_count == 0) server.bindaddr[0] = NULL;
```

遍历全部需要监听的 IP 地址：

```C
for (j = 0; j < server.bindaddr_count || j == 0; j++) {
		// 首个地址为空，监听全部
		if (server.bindaddr[j] == NULL) {
		}
}
```

监听全部 IPv6

```C
fds[*count] = anetTcp6Server(server.neterr,port,NULL,server.tcp_backlog);
```

监听全部 IPv4

```C
fds[*count] = anetTcpServer(server.neterr,port,NULL,server.tcp_backlog);
```

设置为非阻塞模式

```C
anetNonBlock(NULL,fds[*count]);
```

无论是 anetTcp6Server 还是 anetTcpServer 方法，最终都会进入 \_anetTcpServer 函数，通过 af 参数
区分是 IPv4 还是 IPv6。

函数结束后，全部打开的 socket 存储在 server.ipfd 中。

- \_anetTcpServer

获取要监听的地址

```C
struct addrinfo hints, *servinfo, *p;

snprintf(_port,6,"%d",port);
memset(&hints,0,sizeof(hints));
hints.ai_family = af;
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;    /* No effect if bindaddr != NULL */

// 根据 bindaddr 是否为 NULL，获取不同地址信息
if ((rv = getaddrinfo(bindaddr,_port,&hints,&servinfo)) != 0) {
```

遍历全部地址：

```C
for (p = servinfo; p != NULL; p = p->ai_next) {
		// 创建 socket
		if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)
				continue;

		if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;
		// SO_REUSEADDR
		if (anetSetReuseAddr(err,s) == ANET_ERR) goto error;
		if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog) == ANET_ERR) goto error;
		goto end;
}
```

我们来看一下 anetListen 方法

```C
static int anetListen(char *err, int s, struct sockaddr *sa, socklen_t len, int backlog) {
		// 绑定地址
		if (bind(s,sa,len) == -1) {
				anetSetError(err, "bind: %s", strerror(errno));
				close(s);
				return ANET_ERR;
		}

		// listen
		if (listen(s, backlog) == -1) {
				anetSetError(err, "listen: %s", strerror(errno));
				close(s);
				return ANET_ERR;
		}
		return ANET_OK;
}
```

- anetSetBlock

```C
int anetSetBlock(char *err, int fd, int non_block) {
		int flags;

		// 获取 socket 当前配置
		if ((flags = fcntl(fd, F_GETFL)) == -1) {
				anetSetError(err, "fcntl(F_GETFL): %s", strerror(errno));
				return ANET_ERR;
		}

		// 阻塞或非阻塞设置
		if (non_block)
				flags |= O_NONBLOCK;
		else
				flags &= ~O_NONBLOCK;

		// 设置
		if (fcntl(fd, F_SETFL, flags) == -1) {
				anetSetError(err, "fcntl(F_SETFL,O_NONBLOCK): %s", strerror(errno));
				return ANET_ERR;
		}
		return ANET_OK;
}
```

## Accept 事件注册

遍历全部打开的 socket，添加 accept 事件处理

```C
for (j = 0; j < server.ipfd_count; j++) {
		// 注册 accept 事件处理
		if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler,NULL) == AE_ERR)
		{
				// error handler
		}
}
```

来看一下 aeCreateFileEvent 实现

```C
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    // 获取事件
    aeFileEvent *fe = &eventLoop->events[fd];

    // 添加事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    // 更新最大文件句柄
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

- acceptTcpHandler

```C
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);

    // 没有达到一次调用可接受的最大连接数
    while(max--) {
        // 接受连接
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        // 处理连接
        acceptCommonHandler(cfd,0,cip);
    }
}
```

- acceptCommonHandler

```C
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    // 创建新的 client
    if ((c = createClient(fd)) == NULL) {
        // ...
        return;
    }
    // 连接数量超限
    if (listLength(server.clients) > server.maxclients) {
        // ...
        return;
    }

    if (server.protected_mode &&
        server.bindaddr_count == 0 &&
        server.requirepass == NULL &&
        !(flags & CLIENT_UNIX_SOCKET) &&
        ip != NULL)
    {
        // 只接受环回地址请求
        if (strcmp(ip,"127.0.0.1") && strcmp(ip,"::1")) {
            // ...
            return;
        }
    }

    server.stat_numconnections++;
    c->flags |= flags;
}
```

从代码中可以看出，连接被抽象成一个 [Client](client.md) 结构处理。

## References

- [getaddrinfo](https://linux.die.net/man/3/getaddrinfo)
- [socket](https://linux.die.net/man/7/socket)
