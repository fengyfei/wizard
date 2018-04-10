# Controller

## 全景图

![Overview](images/controller-overview.svg)

## 启动的大概过程

![Start Up](images/startup.svg)

## Config

细节请看: [scheme](../general/scheme.md), [flags](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-controller-manager/app/options/options.go#L68:40), [feature](../general/feature.md)

## 启动 HTTP Server

细节请看: [server](../general/server.md)

## 创建 context

![create controller context](images/create-controller-context.svg)

## Informer

这里以 serviceAccountInformer 为例

![service account informer](images/service-account-informer.svg)

细节请看: [informer factory](../client-go/informer_factory.md)
