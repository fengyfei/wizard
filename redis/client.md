# Client

Client 代表一个到 Redis Server 的连接。结构如下图：

![Client](./images/client.png)

## 数据结构

```C
typedef struct client {
    uint64_t id;            // 唯一 ID，递增
    int fd;                 // socket
    redisDb *db;            // 当前选中的 db
    robj *name;             // 名称
    sds querybuf;           // 接收数据缓存
    sds pending_querybuf;   /* If this is a master, this buffer represents the
                               yet not applied replication stream that we
                               are receiving from the master. */
    size_t querybuf_peak;   // 请求峰值
    int argc;               // 当前请求的参数数量
    robj **argv;            // 当前请求参数
    struct redisCommand *cmd, *lastcmd;
    int reqtype;            // 请求协议内容
    int multibulklen;
    long bulklen;
    list *reply;            // 发送给客户端的应答
    unsigned long long reply_bytes; // 应答全部字节数
    size_t sentlen;         // 已发送字节数
    time_t ctime;           // client 创建时间
    time_t lastinteraction;
    time_t obuf_soft_limit_reached_time;
    int flags;              // 标志位
    int authenticated;      // 是否要认证
    int replstate;
    int repl_put_online_on_ack;
    int repldbfd;
    off_t repldboff;
    off_t repldbsize;
    sds replpreamble;
    long long read_reploff;
    long long reploff;
    long long repl_ack_off;
    long long repl_ack_time;
    long long psync_initial_offset;
    char replid[CONFIG_RUN_ID_SIZE+1];
    int slave_listening_port;
    char slave_ip[NET_IP_STR_LEN];
    int slave_capa;
    multiState mstate;
    int btype;
    blockingState bpop;
    long long woff;
    list *watched_keys;
    dict *pubsub_channels;  // pub/sub
    list *pubsub_patterns;
    sds peerid;
    listNode *client_list_node; // client 链表

    int bufpos;                 // 应答缓存
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```

## Create

在 [Connection Management](connection.md) 中我们知道，每一个到 Redis Server 的连接，会被抽象为一个 client 结构，用于后续的处理，在此处，我们来看创建 client 的具体过程。

```C
client *createClient(int fd) {
    // 分配内存
    client *c = zmalloc(sizeof(client));

    // fd 为有效的 socket 时
    if (fd != -1) {
        // 设置 socket 选项
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        // 创建读取 socket 的事件
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }

    // 默认选择 0 号数据库
    selectDb(c,0);
    uint64_t client_id;
    // 生成 client id
    atomicGetIncr(server.next_client_id,client_id,1);
    c->id = client_id;
    c->fd = fd;
    c->name = NULL;
    c->bufpos = 0;
    c->querybuf = sdsempty();
    c->pending_querybuf = sdsempty();
    c->querybuf_peak = 0;
    c->reqtype = 0;
    c->argc = 0;
    c->argv = NULL;
    c->cmd = c->lastcmd = NULL;
    c->multibulklen = 0;
    c->bulklen = -1;
    c->sentlen = 0;
    c->flags = 0;
    c->ctime = c->lastinteraction = server.unixtime;
    c->authenticated = 0;
    c->replstate = REPL_STATE_NONE;
    c->repl_put_online_on_ack = 0;
    c->reploff = 0;
    c->read_reploff = 0;
    c->repl_ack_off = 0;
    c->repl_ack_time = 0;
    c->slave_listening_port = 0;
    c->slave_ip[0] = '\0';
    c->slave_capa = SLAVE_CAPA_NONE;
    c->reply = listCreate();
    c->reply_bytes = 0;
    c->obuf_soft_limit_reached_time = 0;
    listSetFreeMethod(c->reply,freeClientReplyValue);
    listSetDupMethod(c->reply,dupClientReplyValue);
    c->btype = BLOCKED_NONE;
    c->bpop.timeout = 0;
    c->bpop.keys = dictCreate(&objectKeyHeapPointerValueDictType,NULL);
    c->bpop.target = NULL;
    c->bpop.xread_group = NULL;
    c->bpop.xread_consumer = NULL;
    c->bpop.numreplicas = 0;
    c->bpop.reploffset = 0;
    c->woff = 0;
    c->watched_keys = listCreate();
    c->pubsub_channels = dictCreate(&objectKeyPointerValueDictType,NULL);
    c->pubsub_patterns = listCreate();
    c->peerid = NULL;
    c->client_list_node = NULL;
    listSetFreeMethod(c->pubsub_patterns,decrRefCountVoid);
    listSetMatchMethod(c->pubsub_patterns,listMatchObjects);
    if (fd != -1) linkClient(c);
    initClientMultiState(c);
    return c;
}
```

- prepareClientToWrite

```C
int prepareClientToWrite(client *c) {
    // ...

    // 是否已在 pending 列表
    if (!clientHasPendingReplies(c) &&
        !(c->flags & CLIENT_PENDING_WRITE) &&
        (c->replstate == REPL_STATE_NONE ||
         (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
    {
        // 设置标识
        c->flags |= CLIENT_PENDING_WRITE;
        // 放入 pending 列表
        listAddNodeHead(server.clients_pending_write,c);
    }

    return C_OK;
}
```

- addDeferredMultiBulkLength

```C
void *addDeferredMultiBulkLength(client *c) {
    if (prepareClientToWrite(c) != C_OK) return NULL;

    // 链表尾部放入 NULL，作为 multi bulk 的占位符
    listAddNodeTail(c->reply,NULL);
    return listLast(c->reply);
}
```

- addReply

```C
void addReply(client *c, robj *obj) {
    // client 不可执行写操作
    if (prepareClientToWrite(c) != C_OK) return;

    if (sdsEncodedObject(obj)) {
        // 编码方式为 OBJ_ENCODING_RAW 或 OBJ_ENCODING_EMBSTR
        // 首先尝试放入缓存，如果不成功，放入 object list
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyObjectToList(c,obj);
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        // 优先放入发送缓存，避免一次解码操作
        if (listLength(c->reply) == 0 && (sizeof(c->buf) - c->bufpos) >= 32) {
            char buf[32];
            int len;

            len = ll2string(buf,sizeof(buf),(long)obj->ptr);
            if (_addReplyToBuffer(c,buf,len) == C_OK)
                return;
        }

        // 解码对象
        obj = getDecodedObject(obj);
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyObjectToList(c,obj);
        decrRefCount(obj);
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}
```

- writeToClient

```C
int writeToClient(int fd, client *c, int handler_installed) {
    ssize_t nwritten = 0, totwritten = 0;
    size_t objlen;
    sds o;

    // client 有需要发送的数据
    while(clientHasPendingReplies(c)) {
        // 优先发送 buf 中数据
        if (c->bufpos > 0) {
            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
            if (nwritten <= 0) break;

            // 刷新发送计数
            c->sentlen += nwritten;
            totwritten += nwritten;

            // 缓存发送完毕
            if ((int)c->sentlen == c->bufpos) {
                c->bufpos = 0;
                c->sentlen = 0;
            }
        } else {
            // 发送应答对象列表中数据
            o = listNodeValue(listFirst(c->reply));
            objlen = sdslen(o);

            if (objlen == 0) {
                listDelNode(c->reply,listFirst(c->reply));
                continue;
            }

            // 发送
            nwritten = write(fd, o + c->sentlen, objlen - c->sentlen);
            if (nwritten <= 0) break;
            c->sentlen += nwritten;
            totwritten += nwritten;

            // 发送完成
            if (c->sentlen == objlen) {
                // 通过移除头节点指向下一节点
                listDelNode(c->reply,listFirst(c->reply));
                c->sentlen = 0;
                c->reply_bytes -= objlen;
                // 没有剩余对象需要发送
                if (listLength(c->reply) == 0)
                    serverAssert(c->reply_bytes == 0);
            }
        }
        // 单次发送超过设定值，则退出
        // 在 Event Driven Programming 中，避免让一个 event 占用过多的 CPU 时间
        if (totwritten > NET_MAX_WRITES_PER_EVENT &&
            (server.maxmemory == 0 ||
             zmalloc_used_memory() < server.maxmemory) &&
            !(c->flags & CLIENT_SLAVE)) break;
    }

    // 刷新服务器计数
    server.stat_net_output_bytes += totwritten;
    if (nwritten == -1) {
        // 当前忙，可恢复的错误
        if (errno == EAGAIN) {
            nwritten = 0;
        } else {
            serverLog(LL_VERBOSE,
                "Error writing to client: %s", strerror(errno));
            freeClient(c);
            return C_ERR;
        }
    }
    if (totwritten > 0) {
        // slave client，刷新最后交互时间
        if (!(c->flags & CLIENT_MASTER)) c->lastinteraction = server.unixtime;
    }

    // 数据发送完成
    if (!clientHasPendingReplies(c)) {
        c->sentlen = 0;
        // 移除写关注
        if (handler_installed) aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);

        // CLIENT_CLOSE_AFTER_REPLY 设置，则关闭当前 client
        if (c->flags & CLIENT_CLOSE_AFTER_REPLY) {
            freeClient(c);
            return C_ERR;
        }
    }
    return C_OK;
}
```

- processInputBuffer

```C
void processInputBuffer(client *c) {
    // server 当前工作的 client 指向 c
    server.current_client = c;
    
    // querybuf 不为空
    while(sdslen(c->querybuf)) {
        // 非 slave 且暂停状态
        if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;

        // 当前 client 处于 blocked 状态
        if (c->flags & CLIENT_BLOCKED) break;

        // 不要再继续处理请求，因为应答数据会被直接丢弃
        if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

        // 请求类型确定
        if (!c->reqtype) {
            if (c->querybuf[0] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;
            } else {
                c->reqtype = PROTO_REQ_INLINE;
            }
        }

        // 处理各个类型请求，获取命令、参数
        if (c->reqtype == PROTO_REQ_INLINE) {
            if (processInlineBuffer(c) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        if (c->argc == 0) {
            resetClient(c);
        } else {
            // 处理命令
            if (processCommand(c) == C_OK) {
                if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
                    c->reploff = c->read_reploff - sdslen(c->querybuf);
                }

                if (!(c->flags & CLIENT_BLOCKED) || c->btype != BLOCKED_MODULE)
                    resetClient(c);
            }
            if (server.current_client == NULL) break;
        }
    }

    // 处理完毕，重置服务器当前执行 client
    server.current_client = NULL;
}
```

## 辅助方法

- atomicGetIncr

```C
#define atomicGetIncr(var,oldvalue_var,count) do { \
    pthread_mutex_lock(&var ## _mutex); \
    oldvalue_var = var; \
    var += (count); \
    pthread_mutex_unlock(&var ## _mutex); \
} while(0)
```

# References
