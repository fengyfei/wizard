# API Server

## 概览

### 核心结构

![HTTP Server Overview](./images/api_server_overview.svg)

通过将 delegationTarget 注册给服务器链前端的 notFoundHandler，实现顺序访问。

### Hooks

![HTTP Server Hooks](./images/api_server_hooks.svg)

### 配置文件结构

![API Server Config Overview](./images/api_server_config_overview.svg)

## Discovery Group Manager

![Discovery Group Manager Overview](./images/discovery_group_manager.svg)

## 路由安装

### Legacy API 安装

- RESTOptionsGetter 概览：

![API Server Rest Options Getter](./images/api_server_rest_options_getter.svg)

安装路由代码：

```go
// install legacy rest storage
if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
	legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
		StorageFactory:             c.ExtraConfig.StorageFactory,
		ProxyTransport:             c.ExtraConfig.ProxyTransport,
		KubeletClientConfig:        c.ExtraConfig.KubeletClientConfig,
		EventTTL:                   c.ExtraConfig.EventTTL,
		ServiceIPRange:             c.ExtraConfig.ServiceIPRange,
		ServiceNodePortRange:       c.ExtraConfig.ServiceNodePortRange,
		LoopbackClientConfig:       c.GenericConfig.LoopbackClientConfig,
		ServiceAccountIssuer:       c.ExtraConfig.ServiceAccountIssuer,
		ServiceAccountAPIAudiences: c.ExtraConfig.ServiceAccountAPIAudiences,
	}
	m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
}
```

关于 StorageFactory 请看 [Storage](./storage.md) 章节。

- RESTStorageProvider 概览

![REST Storage Provider Overview](./images/rest_storage_provider_overview.svg)

- LegacyRESTStorageProvider 概览

![Legacy REST Storage Provider Overview](./images/legacy_rest_storage_provider_overview.svg)

通过上图，可以看出，通过 NewLegacyRESTStorage 方法，构建出完整的 APIGroupInfo。并与后端存储关联。

- 创建 genericapi.APIGroupVersion

![Create Generic API Group](./images/generic_api_group_info.svg)

```go
func (s *GenericAPIServer) newAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion) *genericapi.APIGroupVersion {
	return &genericapi.APIGroupVersion{
		GroupVersion:     groupVersion,
		MetaGroupVersion: apiGroupInfo.MetaGroupVersion,

		ParameterCodec:  apiGroupInfo.ParameterCodec,
		Serializer:      apiGroupInfo.NegotiatedSerializer,
		Creater:         apiGroupInfo.Scheme,
		Convertor:       apiGroupInfo.Scheme,
		UnsafeConvertor: runtime.UnsafeObjectConvertor(apiGroupInfo.Scheme),
		Defaulter:       apiGroupInfo.Scheme,
		Typer:           apiGroupInfo.Scheme,
		Linker:          apiGroupInfo.GroupMeta.SelfLinker,
		Mapper:          apiGroupInfo.GroupMeta.RESTMapper,

		Admit:                        s.admissionControl,
		Context:                      s.RequestContextMapper(),
		MinRequestTimeout:            s.minRequestTimeout,
		EnableAPIResponseCompression: s.enableAPIResponseCompression,
	}
}
```

APIGroupVersion 生成后，直接安装路由：

```go
if err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer); err != nil {
```