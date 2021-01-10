# Installation

Iris是一个跨平台框架。

运行只需要go语言的环境, 要求版本在1.14及以上。

```bash
go env -w GO111MODULE=on
```

## Install

```bash
go get github.com/kataras/iris/v12@v12.1.8
```

或者可以编辑你项目的`go.mod`文件。

```go
module your_project_name

go 1.14

require (
    github.com/kataras/iris/v12 v12.1.8
)
```

```shell
go build
```

## TroubleShooting

如果你在安装iris的时候遇到了错误, 请确保你的[`GOPROXY`环境变量](https://github.com/golang/go/wiki/Modules#are-there-always-on-module-repositories-and-enterprise-proxies)设置为一个有效的值

```bash
go env -w GOPROXY=https://goproxy.cn,https://gocenter.io,https://goproxy.io,direct
```
