# Host

## 监听与服务

你可以启动服务(们)监听任何形式的`net.Listener`甚至`http.Server`实例。服务初始化的方法应该被放置在最后, 通过`Run`函数实现。

Go开发者最常用启动他们服务的方法, 是通过"主机名:ip"方式的网络连接实现的。在Iris中, 我们使用`iris.Runner`类型的`iris.Addr`

```go
import "github.com/kataras/iris/v12"
```

```go
app := iris.New()
// tcp监听 0.0.0.0:8080

// Listen是app.Run(iris.Addr(":8080"))的简写
app.Listen(":8080")
```

有时候你选择完全覆盖控制`http.Server`实例

```go
import "net/http"
```

```go
// 与上述相同, 不过使用了自定义的 http.Server, 可能在一些地方也会用到
server := &http.Server{Addr: ":8080" /* 其它 */}
app.Run(iris.Server(server))
```

最高级的用法是创建一个自定义的`net.Listener`并将其传递给`app.Run`

```go
// 使用自定义的net.Listener
l, err := net.Listen("tcp4", ":8080")
if err != nil {
    panic(err)
}
app.Run(iris.Listener(l))
```

一个更完整的示例如下, 使用了unix独有的socket文件特性

```go
package main

import (
    "os"
    "net"

    "github.com/kataras/iris/v12"
)

func main() {
    app := iris.New()

    // Unix socket
    if errOs := os.Remove(socketFile); errOs != nil && !os.IsNotExist(errOs) {
        app.Logger().Fatal(errOS)
    }

    l, err := net.Listen("unix", socketFile)

    if err != nil {
        app.Logger().Fatal(err)
    }

    if err = os.Chmod(socketFile, mode); err != nil {
        app.Logger().Fatal(err)
    }

    app.Run(iris.Listener(l))
}
```

UNIX和BSD主机可以发挥复用端口的优势

```go
package main

import (
    // 包 tcplisten 提供了可自定义的 TCP net.Listener 并伴随了大量的性能相关的选项
    //
    // - SO_REUSEPORT. 这个选项支持在多CPU服务器上线性缩放服务的性能
    // 参考 https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/ 获取细节
    //
    // - TCP_DEFER_ACCEPT. 这个选项在向其写入之前预估接受到的请求
    //
    // - TCP_FASTOPEN. 参考 https://lwn.net/Articles/508865/ 获取细节
    "github.com/valyala/tcplisten"

    "github.com/kataras/iris/v12"
)

// go get github.com/valyala/tcplisten
// go run main.go
func main() {
    app := iris.New()

    app.Get("/", func(ctx iris.Context) {
        ctx.HTML("<h1>Hello World!</h1>")
    })

    listenerCfg := tcplisten.Config{
        ReusePort:   true,
        DeferAccept: true,
        FastOpen:    true,
    }

    l, err := listenerCfg.NewListener("tcp", ":8080")
    if err != nil {
        app.Logger().Fatal(err)
    }

    app.Run(iris.Listener(l))
}
```

## http/2与安全

如果你有本地的认证和服务器密钥文件, 你可以使用`iris.TLS`去启动``https://`服务

```go
// TLS 使用文件挥着原内容
app.Run(iris.TLS("127.0.0.1:443", "mycert.cert", "mykey.key"))
```

当你准备把你的应用程序向外开放时, 你应当使用的方法是`iris.AutoTLS`, 一个启动安全服务的方法, 由[https://letsencrypt.org/](https://letsencrypt.org/)提供**免费**认证

```go
// 自动TLS
app.Run(iris.AutoTLS(":443", "example.com", "admin@example.com"))
```

## 一些 `iris.Runner`

有时候你看看你想要一些特殊的监听, 可能并不是`net.Listener`类型。你可以通过`iris.Raw`来实现, 但是你需要为这些方法负责

```go
// 使用一些func() 错误,
//
```
