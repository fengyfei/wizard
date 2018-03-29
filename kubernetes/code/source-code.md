# 源码

## 拉去源码

应将源码放到 `$GOPATH/src/k8s.io` 目录下

```shell
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone git@github.com:kubernetes/kubernetes.git
# 四种 git clone 任选一个
# git clone https://github.com/kubernetes/kubernetes.git
# 库比较大，可以只拉取最新的代码
# git clone git@github.com:kubernetes/kubernetes.git --depth=1
# git clone https://github.com/kubernetes/kubernetes.git --depth=1
```

## 编译源码

千万不要使用 `go build`，所有编译在 `k8s.io/kubernetes` 目录下使用 `make` 进行，例：`make release`、`make kube-apiserver`等，编译好的文件在 `k8s.io/kubernetes/_output` 目录下。