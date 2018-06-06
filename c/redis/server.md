# Redis Server

## 启动

保存启动路径

```C
server.executable = getAbsolutePath(argv[0]);
server.exec_argv = zmalloc(sizeof(char*)*(argc+1));
server.exec_argv[argc] = NULL;
```

检查配置文健，如果是哨兵模式，执行哨兵服务初始化

```C
if (server.sentinel_mode) {
    initSentinelConfig();
    // 命令列表仅包含哨兵模式下命令
    initSentinel();
}
```

检查存储模式(RDB/AOF)设定

```C
if (strstr(argv[0],"redis-check-rdb") != NULL)
    redis_check_rdb_main(argc,argv,NULL);
else if (strstr(argv[0],"redis-check-aof") != NULL)
    redis_check_aof_main(argc,argv);
```

获取 supervise 模式设置

```C
server.supervised = redisIsSupervised(server.supervised_mode);
```

获取后台运行设置，并后台运行

```C
int background = server.daemonize && !server.supervised;
if (background) daemonize();
```

前序检查、设置完毕，开启初始化服务器，首先，设置信号量

```C
signal(SIGHUP, SIG_IGN);
signal(SIGPIPE, SIG_IGN);
setupSignalHandlers();
```

初始化服务器基本配置

```C
server.pid = getpid();
server.current_client = NULL;
server.clients = listCreate();
server.clients_to_close = listCreate();
server.slaves = listCreate();
server.monitors = listCreate();
server.clients_pending_write = listCreate();
server.slaveseldb = -1; /* Force to emit the first SELECT command. */
server.unblocked_clients = listCreate();
server.ready_keys = listCreate();
server.clients_waiting_acks = listCreate();
server.get_ack_from_slaves = 0;
server.clients_paused = 0;
server.system_memory_size = zmalloc_get_memory_size();
```

调整打开最大句柄数

```C
adjustOpenFilesLimit()
```

初始化 Event Loop

```C
server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
```

创建数据库

```C
server.db = zmalloc(sizeof(redisDb)*server.dbnum);
```

开启 TCP 监听

```C
if (server.port != 0 &&
    listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
    exit(1);
```

开启 Unix Socket 监听

```C
if (server.unixsocket != NULL) {
    unlink(server.unixsocket); /* don't care if this fails */
    server.sofd = anetUnixServer(server.neterr,server.unixsocket,
        server.unixsocketperm, server.tcp_backlog);
    if (server.sofd == ANET_ERR) {
        serverLog(LL_WARNING, "Opening Unix socket: %s", server.neterr);
        exit(1);
    }
    anetNonBlock(NULL,server.sofd);
}
```

初始化数据库

```C
for (j = 0; j < server.dbnum; j++) {
    server.db[j].dict = dictCreate(&dbDictType,NULL);
    server.db[j].expires = dictCreate(&keyptrDictType,NULL);
    server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
    server.db[j].ready_keys = dictCreate(&objectKeyPointerValueDictType,NULL);
    server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
    server.db[j].id = j;
    server.db[j].avg_ttl = 0;
}
```

LRU Pool 创建

```C
evictionPoolAlloc()
```

Pub/Sub 模式初始化

```C
server.pubsub_channels = dictCreate(&keylistDictType,NULL);
server.pubsub_patterns = listCreate();
listSetFreeMethod(server.pubsub_patterns,freePubsubPattern);
listSetMatchMethod(server.pubsub_patterns,listMatchPubsubPattern);
```

AOF 初始化

```C
aofRewriteBufferReset();
```

更新缓存时间

```C
updateCachedTime();
```

创建定时任务定时器

```C
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    serverPanic("Can't create event loop timers.");
    exit(1);
}
```

创建接受连接 Handler 事件

```C
for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
        acceptTcpHandler,NULL) == AE_ERR)
        {
            serverPanic(
                "Unrecoverable error creating server.ipfd file event.");
        }
}
```

打开 AOF 文件

```C
if (server.aof_state == AOF_ON) {
    server.aof_fd = open(server.aof_filename,
                           O_WRONLY|O_APPEND|O_CREAT,0644);
    if (server.aof_fd == -1) {
        serverLog(LL_WARNING, "Can't open the append-only file: %s",
            strerror(errno));
        exit(1);
    }
}
```

集群初始化

```C
if (server.cluster_enabled) clusterInit();
```

Script 缓存初始化

```C
replicationScriptCacheInit()
```

lua 脚本引擎初始化

```C
scriptingInit()
```

slowlog 初始化

```C
slowlogInit()
```

延迟监控初始化

```C
latencyMonitorInit()
```

初始化后台系统

```C
bioInit()
```

启动 Event Loop

```C
aeMain(server.el);
```

### 关键代码

- Linux 后台运行代码

```C
void daemonize(void) {
    int fd;

    if (fork() != 0) exit(0); /* parent exits */
    setsid(); /* create a new session */

    /* Every output goes to /dev/null. If Redis is daemonized but
     * the 'logfile' is set to 'stdout' in the configuration file
     * it will not log at all. */
    if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > STDERR_FILENO) close(fd);
    }
}
```

## Reference

- [Redis Persistence](https://redis.io/topics/persistence)
