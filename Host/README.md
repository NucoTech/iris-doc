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

如果你有本地的认证和服务器密钥文件, 你可以使用`iris.TLS`去启动`https://`服务

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
// 你需要注意通过这种方式启动监听责任,
// 为了使用这些朴素的特性, 我们将使用 `net/http` 包的ListenAndServe函数
srv := &http.Server{Addr: ":8080"}
app.Run(iris.Raw(srv.ListenAndServe))
```

## Host配置项

上述所有形式的监听接受一个`func(*iris.Supervisor)`的仅存的、可变的参数, 这被用于给那些你通过的这类方法指定的host添加配置

举个例子, 我们想那些被我们干掉的服务, 在他们关机时给你一个回调

```go
app.Run(iris.Addr(":8080", func(h *iris.Supervisor) {
    h.RegisterOnShutdown(func() {
        println("server terminated")
    })
}))
```

你甚至可以在`app.Run`方法之前做这些, 但是不同的是这些host配置会作用于你所启动web应用程序的所有host(通过`app.NewHost` 我们等会就可以看到这些)

```go
app := iris.New()
app.ConfigureHost(func(h *iris.Supervisor) {
    h.RegisterOnShutdown(func() {
        println("server terminated")
    })
})
app.Listen(":8080")
```

在`Run`方法之后, 你应用程序部署的所有host的权限由`Application#Hosts`域提供

但是最常见的场景是你可能需要在`app.Run`方法之前获得host的权限, 这儿有两种获得host supervisor权限的方法, 继续往下看

我们上面就知晓了如何通过`app.Run`或者`app.ConfigureHost`的第二个参数配置所有的应用程序host。这有另一个方法对简单场景更合适, 即使用`app.NewHost`去创建一个新的host并且使用它的`Server`或`Listen`函数之一通过`iris#Raw`启动器请求应用程序

注意这种方法需要引入额外的`net/http`包

示例代码

```go
h := app.NewHost(&http.Server{Addr: ":8080"})
h.RegisterOnShutdown(func() {
    println("server terminated")
})

app.Run(iris.Raw(h.ListenAndServe))
```

## 多host

你可以在多个服务器上部署iris web应用, `iris.Router`与`net/http/Handler`函数是兼容的, 你可以理解为它可以适配任何`net/http`服务。然而吧，这有个更简单的方法，通过使用`app.NewHost`来实现。它也复制了所有host配置器, 可以让你通过web app的`app.Shutdown`关闭所有的host

```go
app := iris.New()
app.Get("/", indexHandler)

// 为了不占用主进程的 "goroutine" 跑在不同的 goroutine上
go app.Listen(":8080")
// 启动第二个服务tcp监听 0.0.0.0:9090,
// 没有使用 "go" 关键字因为我们准备在最后一个服务的运行时干掉
app.NewHost(&http.Server{Addr: ":9090"}).ListenAndServe()
```

## 关机(优雅点)

让我们继续学习如何通过捕获 `Ctrl + C / Command + C`或者unix的`kill`命令, 来优雅的干掉服务

> 使用`Ctrl + C / Command + C`或者unix的`kill`命令来优雅关机是**默认支持**的

为了手动管理当应用挂掉做的事, 我们应该使用`WithoutInterruptHandler`配置项禁掉默认的动作, 然后注册一个新的处理器(全局的, 横跨所有可能的host)

示例代码

```go
package main

import (
    "context"
    "time"

    "github.com/kataras/iris/v12"
)

func main() {
    app := iris.New()

    iris.RegisterOnInterrupt(func() {
        timeout := 5 * time.Second
        ctx, cancel := context.WithTimeout(context.Background(), timeout)
        defer cancel()
        // 关闭所有的host
        app.Shutdown(ctx)
    })

    app.Get("/", func(ctx iris.Context) {
        ctx.HTML("<h1>那个, 我只是为了看看服务有没有被干掉的~~</h1>")
    })

    app.Listen(":8080", iris.WithoutInterruptHandler)
}
```

继续阅读[配置项](../Configuration.md)这一节了解更多关于`app.Run`的第二个可变参数
