# Cobra

Cobra 是用于创建 CLI 应用程序的库，同时提供了用于生成应用程序和命令文件的程序。

非常多有名的开源项目都在使用它：

[Kubernetes](http://kubernetes.io/)、[Hugo](http://gohugo.io)、[rkt](https://github.com/coreos/rkt)、[etcd](https://github.com/coreos/etcd)、[Moby (former Docker)](https://github.com/moby/moby)、[Docker (distribution)](https://github.com/docker/distribution)、[OpenShift](https://www.openshift.com/)、[Delve](https://github.com/derekparker/delve)、[GopherJS](http://www.gopherjs.org/)、[CockroachDB](http://www.cockroachlabs.com/)、[Bleve](http://www.blevesearch.com/)、[ProjectAtomic (enterprise)](http://www.projectatomic.io/)、[GiantSwarm's swarm](https://github.com/giantswarm/cli)、[Nanobox](https://github.com/nanobox-io/nanobox)/[Nanopack](https://github.com/nanopack)、[rclone](http://rclone.org/)、[nehm](https://github.com/bogem/nehm)、[Pouch](https://github.com/alibaba/pouch)

## Example

这里有一个多层嵌套的例子，[example](https://github.com/spf13/cobra/blob/master/README.md#example)：

```go
// basic.go
package main

import (
    "fmt"
    "strings"

    "github.com/spf13/cobra"
    "github.com/spf13/pflag"
)

func main() {
    var echoTimes int

    var cmdPrint = &cobra.Command{
        Use:   "print [string to print]",
        Short: "Print anything to the screen",
        Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
        Args: cobra.MinimumNArgs(1),
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("Print: " + strings.Join(args, " "))
            flags := cmd.Flags()
            flags.VisitAll(func(flag *pflag.Flag) {
                fmt.Printf("FLAG: --%s=%q\n", flag.Name, flag.Value)
            })
        },
    }

    var cmdEcho = &cobra.Command{
        Use:   "echo [string to echo]",
        Short: "Echo anything to the screen",
        Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
        Args: cobra.MinimumNArgs(1),
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("Print: " + strings.Join(args, " "))
            flags := cmd.Flags()
            flags.VisitAll(func(flag *pflag.Flag) {
                fmt.Printf("FLAG: --%s=%q\n", flag.Name, flag.Value)
            })
        },
    }

    var cmdTimes = &cobra.Command{
        Use:   "times [# times] [string to echo]",
        Short: "Echo anything to the screen more times",
        Long: `echo things multiple times back to the user by providing
a count and a string.`,
        Args: cobra.MinimumNArgs(1),
        Run: func(cmd *cobra.Command, args []string) {
            for i := 0; i < echoTimes; i++ {
                fmt.Println("Echo: " + strings.Join(args, " "))
            }
            flags := cmd.Flags()
            flags.VisitAll(func(flag *pflag.Flag) {
                fmt.Printf("FLAG: --%s=%q\n", flag.Name, flag.Value)
            })
        },
    }

    cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

    var rootCmd = &cobra.Command{Use: "app"}
    rootCmd.AddCommand(cmdPrint, cmdEcho)
    cmdEcho.AddCommand(cmdTimes)
    rootCmd.Execute()
}
```

```shell
$go build -o app basic.go
$./app echo 123456
Print: 123456
FLAG: --help="false"
$./app echo 123456
Print: 123456
FLAG: --help="false"
./app echo times 123456 -t 3
Echo: 123456
Echo: 123456
Echo: 123456
FLAG: --help="false"
FLAG: --times="3"
```

## Kubernetes

```go
    // 默认的标志类参数，用于在后面设置
    s := options.NewKubeControllerManagerOptions()
    command := &cobra.Command{
        Use: "kube-xxx",
        Long: ``,
        Run: func(cmd *cobra.Command, args []string) {
            // 有 -version 参数，输出并退出
            verflag.PrintAndExitIfRequested()
            // 打印日志
            utilflag.PrintFlags(cmd.Flags())

            if err := Run(c.Complete()); err != nil {
                fmt.Fprintf(os.Stderr, "%v\n", err)
                os.Exit(1)
            }
        },
    }
    // 给 cmd 设置标志类参数
    s.AddFlags(cmd.Flags(), KnownControllers(), ControllersDisabledByDefault.List())

    // 设置正常化函数处理参数下标
    pflag.CommandLine.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)
    // 将 goflag.CommandLine 里的参数添加到 pflag.CommandLine 中
    pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)

    logs.InitLogs()
    defer logs.FlushLogs()

        // 运行
        if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
```

什么是标志类参数？

* [https://golang.org/pkg/flag/](https://golang.org/pkg/flag/)
* [https://gobyexample.com/command-line-flags](https://gobyexample.com/command-line-flags)