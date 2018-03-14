# Docker Networking

## 创建

dockerd 的 netController 在 daemon.restore() 方法中创建：

```go
daemon.netController, err = daemon.initNetworkController(daemon.configStore, activeSandboxes)
```

## References

- [Network Overview](https://docs.docker.com/network/)
- [Networking with standalone containers](https://docs.docker.com/network/network-tutorial-standalone/)
- [Networking using the host network](https://docs.docker.com/network/network-tutorial-host/)
- [Networking with overlay networks](https://docs.docker.com/network/network-tutorial-overlay/)
- [Networking using a macvlan network](https://docs.docker.com/network/network-tutorial-macvlan/)