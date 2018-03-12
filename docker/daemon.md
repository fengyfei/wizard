# docker 结构

Docker 采用 C/S 模型:

![Docker Architecture](https://docs.docker.com/engine/images/architecture.svg)

其中，dockerd 通过监听 docker client 的请求，来构建、运行、分发 docker 容器。

## dockerd 启动过程

### 概览

dockerd 是基于命令行工具 [cobra](https://github.com/spf13/cobra) 来构建的：

```go
cmd := &cobra.Command{
	Use:           "dockerd [OPTIONS]",
	Short:         "A self-sufficient runtime for containers.",
	SilenceUsage:  true,
	SilenceErrors: true,
	Args:          cli.NoArgs,
	RunE: func(cmd *cobra.Command, args []string) error {
		opts.flags = cmd.Flags()
		return runDaemon(opts)
	},
}
```

在 runDaemon 中，创建一个 DaemonCli 示例，并运行该实例的 start 方法，启动 dockerd。 DaemonCli 定义如下：

```go
type DaemonCli struct {
	*config.Config
	configFile *string
	flags      *pflag.FlagSet

	api             *apiserver.Server // HTTP 服务
	d               *daemon.Daemon
	authzMiddleware *authorization.Middleware // 授权中间件
}
```

### DaemonCli.start 方法

- 设置默认配置参数，并加载配置:

```go
opts.SetDefaultOptions(opts.flags)

if cli.Config, err = loadDaemonCliConfig(opts); err != nil {
	return err
}
cli.configFile = &opts.configFile
cli.flags = opts.flags

if cli.Config.Debug {
	debug.Enable()  // 通过设置： os.Setenv("DEBUG", "1") 开启调试
}
```

- 设置 umask 并创建 pid 文件

```go
if err := daemon.CreateDaemonRoot(cli.Config); err != nil {
	return err
}

if cli.Pidfile != "" {
	pf, err := pidfile.New(cli.Pidfile)

	defer func() {
		if err := pf.Remove(); err != nil {
			logrus.Error(err)
		}
	}()
}
```

- 构建 APIServerConfig 并启动 APIServer:

```go
cli.api = apiserver.New(serverConfig)
```

解析监听地址，并创建 listener：

```go
for i := 0; i < len(cli.Config.Hosts); i++ {
	// ...
	// proto: fd | tcp | unix
        ls, err := listeners.Init(proto, addr, serverConfig.SocketGroup, serverConfig.TLSConfig)
	if err != nil {
		return err
	}
	ls = wrapListeners(proto, ls)
	// 对于 tcp 协议，需要保证监听的端口不在 container 可使用的端口区间中
	if proto == "tcp" {
		if err := allocateDaemonPort(addr); err != nil {
			return err
		}
	}
	// ...
	hosts = append(hosts, protoAddrParts[1])
	cli.api.Accept(addr, ls...)
}
```

在代码最后，```go cli.api.Accept ``` 创建一组 http.Server:

```go
func (s *Server) Accept(addr string, listeners ...net.Listener) {
	for _, listener := range listeners {
		httpServer := &HTTPServer{
			srv: &http.Server{
				Addr: addr,
			},
			l: listener,
		}
		s.servers = append(s.servers, httpServer)
	}
}
```

要注意，在 Accept 中，只是创建了 http.Server 结构，但是并没有开始处理请求。

- 创建 Registry 服务

```go
registryService, err := registry.NewService(cli.Config.ServiceOptions)
```

- 启动 containerd

```go
// 根据 host 操作系统类型，初始化 containerd 配置
rOpts, err := cli.getRemoteOptions()
// 创建 containerd 实例
// 根据配置不同，在此处 containerd 有可能已经启动
containerdRemote, err := libcontainerd.New(filepath.Join(cli.Config.Root, "containerd"), filepath.Join(cli.Config.ExecRoot, "containerd"), rOpts...)
```

- 插件初始化

```go
pluginStore := plugin.NewStore()

if err := cli.initMiddlewares(cli.api, serverConfig, pluginStore); err != nil {
	logrus.Fatalf("Error creating middlewares: %v", err)
}
```

- 初始化 daemon.Daemon

```go
d, err := daemon.NewDaemon(cli.Config, registryService, containerdRemote, pluginStore)
if err != nil {
	return fmt.Errorf("Error starting daemon: %v", err)
}

d.StoreHosts(hosts)
```

- cluster 初始化

```go
watchStream := make(chan *swarmapi.WatchMessage, 32)

c, err := cluster.New(cluster.Config{
	Root:                   cli.Config.Root,
	Name:                   name,
	Backend:                d,
	ImageBackend:           d.ImageService(),
	PluginBackend:          d.PluginManager(),
	NetworkSubnetsProvider: d,
	DefaultAdvertiseAddr:   cli.Config.SwarmDefaultAdvertiseAddr,
	RuntimeRoot:            cli.getSwarmRunRoot(),
	WatchStream:            watchStream,
})
// ...
d.SetCluster(c)
err = c.Start()
```

- 设置路由并启动 http.Server

```go
routerOptions, err := newRouterOptions(cli.Config, d)
// ...
routerOptions.api = cli.api
routerOptions.cluster = c

initRouter(routerOptions)

// process cluster change notifications
watchCtx, cancel := context.WithCancel(context.Background())
defer cancel()
go d.ProcessClusterNotifications(watchCtx, watchStream)

// ...

serveAPIWait := make(chan error)
go cli.api.Wait(serveAPIWait)
```

至此，dockerd 启动完成。

## References
- [docker architecture](https://docs.docker.com/engine/docker-overview/#docker-architecture)
- [dockerd](https://docs.docker.com/engine/reference/commandline/dockerd/)
- [coreos/systemd](https://github.com/coreos/go-systemd)
