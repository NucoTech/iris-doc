# 下载Iris(Installation)

Iris是一个跨平台框架。  
运行需要go语言的环境, 要求版本在1.14及以上。

```shell
go env -w GO111MODULE=on
```

## 安装(Install)

```shell
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

## 常见问题(TroubleShooting)

如果你在安装`iris`的时候遇到了错误, 请确保你的[`GOPROXY`环境变量](https://github.com/golang/go/wiki/Modules#are-there-always-on-module-repositories-and-enterprise-proxies)设置为一个有效的值

```shell
go env -w GOPROXY=https://goproxy.cn,https://gocenter.io,https://goproxy.io,direct
```
