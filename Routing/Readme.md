# 路由

## 类型处理

顾名思义，类型处理用来处理程序不同类型的请求。

处理程序响应HTTP请求，将应答头和数据写入`Context.ResponseWriter()` 并将其返回。返回标志着这个请求的结束；在程序调用结束之后将无法继续使用`Context`的内容。

不管是HTTP客户端，HTTP协议版本，还有任何存在于客户端和iris服务之间的中间件，在结束`context.ResponseWriter()`之后将再也不被允许通过`Context.Request().Body`读取。所以警示处理器应该首先读取`Context.Request().Body`之后再进行请求回复。

除了读取消息正文，处理器并不被允许修改提供的`Context`。

如果有一个处理器崩溃，服务器 (处理器的调用者) 首先假定处理器崩溃的影响范围，并将它们隔离。它将重新恢复崩溃的处理器，并将堆栈跟踪记录到服务器错误日志并断开连接。

```go
type Handler func(iris.Context)
```

一旦一个处理器被注册，我们可以使用返回的Route实例为处理器提供一个名称，以便于调试或匹配视图中的相对路径。有关更多信息，请查看[反向查找](/ReverseLookups.md)部分。

## 行为

Iris的默认行为是使用`/api/user`这样的路径接受和注册路由,没有尾随斜杠。如果客户机试图访问`$your\u host/api/user/`则Iris路由器将自动永久重定向到`$your\u host/api/user`，以便由注册路由处理。这是设计api的现代方法。

但是，如果要对请求的资源禁用路径更正你可以通过配置iris配置中的 `iris.WithoutPathCorrection` 选项到你的`app.Run`，例如：

```go
// [app := iris.New...]
// [...]

app.Listen(":8080", iris.WithoutPathCorrection)
```

如果你想在`/adi/user`和`/api/user/`两个路径的路由中保持控制器的一致而不是通过重定向，你只需要通过`iris.WithoutPathCorrectionRedirection`选项来替代：

```go
app.Listen(":8080", iris.WithoutPathCorrectionRedirection)
```

## API

支持所有类型的Http方法，开发者可以在同一个路径下绑定多个不同的Http方法。

第一个参数是Http方法，第二个参数是路由的请求路径，第三个可变参数应该包含至少一个`iris.Handler`，当客户端从服务器请求特定的资源路径时，根据绑定的顺序逐一执行。

示例代码:

```go
app := iris.New()

app.Handle("GET", "/contact", func(ctx iris.Context) {
    ctx.HTML("<h1> Hello from /contact </h1>")
})
```

为了使最终开发者的工作更加简单，iris对每种Http方法都提供了帮助方法，第一个参数是路由的请求路径，第二个可变参数需要包含至少一个`iris.Handler`，当客户端从服务器请求特定的资源路径时，根据绑定的顺序逐一执行。

示例代码：

```go
app := iris.New()

// Method: "GET"
app.Get("/", handler)

// Method: "POST"
app.Post("/", handler)

// Method: "PUT"
app.Put("/", handler)

// Method: "DELETE"
app.Delete("/", handler)

// Method: "OPTIONS"
app.Options("/", handler)

// Method: "TRACE"
app.Trace("/", handler)

// Method: "CONNECT"
app.Connect("/", handler)

// Method: "HEAD"
app.Head("/", handler)

// Method: "PATCH"
app.Patch("/", handler)

// register the route for all HTTP Methods
app.Any("/", handler)

func handler(ctx iris.Context){
    ctx.Writef("Hello from method: %s and path: %s\n", ctx.Method(), ctx.Path())
}
```

### 离线路由

你也可以使用Iris里面的一个特殊方法，它叫做`None`，你可是使用它去从外部隐藏一个路由但是仍然可以由其他路由的控制器通过`Context.Exec`方法来调用。每个API控制器方法返回一个路由值。可以通过`IsOnline`方法来返回路由现在的状态。你可以通过一个路由的`Route.Method`值将这个路由的状态由离线改为在线，反之亦然。当然，在服务器环境下需要一个可以安全使用的`app.RefreshRouter()`才能对路由进行修改。让我们看一个更完整的实例代码：

```go
// file: main.go
package main

import (
    "github.com/kataras/iris/v12"
)

func main() {
    app := iris.New()

    none := app.None("/invisible/{username}", func(ctx iris.Context) {
        ctx.Writef("Hello %s with method: %s", ctx.Params().Get("username"), ctx.Method())

        if from := ctx.Values().GetString("from"); from != "" {
            ctx.Writef("\nI see that you're coming from %s", from)
        }
    })

    app.Get("/change", func(ctx iris.Context) {

        if none.IsOnline() {
            none.Method = iris.MethodNone
        } else {
            none.Method = iris.MethodGet
        }

        // refresh re-builds the router at serve-time in order to
        // be notified for its new routes.
        app.RefreshRouter()
    })

    app.Get("/execute", func(ctx iris.Context) {
        if !none.IsOnline() {
            ctx.Values().Set("from", "/execute with offline access")
            ctx.Exec("NONE", "/invisible/iris")
            return
        }

        // same as navigating to "http://localhost:8080/invisible/iris"
        // when /change has being invoked and route state changed
        // from "offline" to "online"
        ctx.Values().Set("from", "/execute")
        // values and session can be
        // shared when calling Exec from a "foreign" context.
        // 	ctx.Exec("NONE", "/invisible/iris")
        // or after "/change":
        ctx.Exec("GET", "/invisible/iris")
    })

    app.Listen(":8080")
}
```

#### 如何去运行

1. `go run main.go`
2. 通过浏览器打开` http://localhost:8080/invisible/iris ` 你将看到一个 `404 not found` 页面
3. 然而 `http://localhost:8080/execute` 的路由能够被执行
4. 现在，如果你打开 `http://localhost:8080/change` 并刷新 `/invisible/iris` 标签，你将看到你能看到的。
