# Option Function

用于设置配置文件时，同时不需要暴露配置文件具体内容。

## 详细说明

首先定义 Config 结构

```go
type Config struct {
	// ...
}
```

然后，定义 Option 类型：

```go
type Option func(c *Config)
```

然后，可以通过闭包方式，返回每一个具体的 Option 方法：

```go
func OptionDefault(dn string) Option {
	return func(c *Config) {
		// ...
	}
}
```

要注意，返回 Option 的方法一般要大写，这样，就可以将配置文件的细节隐藏起来；

外部使用者，只需要看到产生 Option 的方法即可。

如果要改变配置文件内容，外部应用可以做到不改动或很少改动。

## References

- [libnetwork](https://github.com/docker/libnetwork)
