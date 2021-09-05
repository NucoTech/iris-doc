# 路由中间件(Routing middleware)

当我们在Iris中谈论中间件时, 我们谈论的是在HTTP请求生命周期中主处理程序代码之前和/或之后运行代码。例如, 日志中间件可能会将传入的请求详细信息写入日志, 然后在写入关于日志响应的详细信息之前调用处理程序代码。中间件很酷的一点是, 这些单元非常灵活并且可重用

中间件是一个 `func(ctx iris.Context)`**处理程序**形式, 当牵一个中间件调用 `ctx.Next()`时,中间件正在执行,这可以用于身份验证, 即:是否请求身份验证然后调用 `ctx.Next()` 来处理请求中的处理程序链的其余部分, 否则将触发错误响应

## 写一个中间件

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()
    // 或者 app.Use(before) 和 app.Done(after).
    app.Get("/", before, mainHandler, after)
    app.Listen(":8080")
}

func before(ctx iris.Context) {
    shareInformation := "this is a sharable information between handlers"

    requestPath := ctx.Path()
    println("Before the mainHandler: " + requestPath)

    ctx.Values().Set("info", shareInformation)
    ctx.Next() // 执行下一个处理程序, 在本例中是主处理程序
}

func after(ctx iris.Context) {
    println("After the mainHandler")
}

func mainHandler(ctx iris.Context) {
    println("Inside mainHandler")

    // 从 "before" 处理程序中获取信息
    info := ctx.Values().GetString("info")

    // 向客户端写入一些内容作为响应
    ctx.HTML("<h1>Response</h1>")
    ctx.HTML("<br/> Info: " + info)

    ctx.Next() // 执行 "after".
}
```

```shell
$ go run main.go # 并导航到 http://localhost:8080
Now listening on: http://localhost:8080
Application started. Press CTRL+C to shut down.
Before the mainHandler: /
Inside mainHandler
After the mainHandler
```

## 全局中间件

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    // 注册我们的路由
    app.Get("/", indexHandler)
    app.Get("/contact", contactHandler)

    // 这些调用的顺序无关紧要, "UseGlobal" 和 "DoneGlobal" 
    // 也会应用于现有的路由和未来的路由

    // 请记住: "Use" 和  "Done" 是应用到当前Party及其子Party的, 
    // 所以如果我们在路由注册之前使用 "app.Use/Done", 
    // 它会像 UseGlobal/DoneGlobal 一样工作, 因为 "app" 是根 "Party"
    app.UseGlobal(before)
    app.DoneGlobal(after)

    app.Listen(":8080")
}

func before(ctx iris.Context) {
     // [...]
}

func after(ctx iris.Context) {
    // [...]
}

func indexHandler(ctx iris.Context) {
    // 向客户端写入一些内容作为响应
    ctx.HTML("<h1>Index</h1>")

    ctx.Next() // 执行通过 "Done" 注册的 "after" 处理程序
}

func contactHandler(ctx iris.Context) {
    // 向客户端写入一些内容作为响应
    ctx.HTML("<h1>Contact</h1>")

    ctx.Next() // 执行通过 "Done" 注册的 "after" 处理程序
}
```

您还可以使用 `ExecutionRules` 强制执行已完成的处理程序, 而不需要在路由处理程序中使用 `ctx.Next()`, 如下所示

```go
app.SetExecutionRules(iris.ExecutionRules{
    // Begin: ...
    // Main:  ...
    Done: iris.ExecutionOptions{Force: true},
})
```

## 转换 `http.Handler/HandlerFunc`

然而, 您并不局限于它们——您可以自由地使用任何与 [net/http](https://golang.org/pkg/net/http/) 包兼容的第三方中间件

Iris, 不像其他的, 是100%兼容的标准, 这就是为什么大多数大公司适应他们的工作流程, 像一个非常著名的美国电视网络, 相信Iris;它是最新的, 它将始终与std `net/http` 包保持一致, 该包是由Go的作者在每个新的Go编程语言版本上现代化的

任何为 `net/http` 编写的第三方中间件都可以使用 `Iris.fromstd`(一个第三方Party中间件)与Iris兼容。记住, `ctx.ResponseWriter()` 和 `ctx.Request()` 返回与 [http.Handler](https://golang.org/pkg/net/http/#Handler) 相同的 `net/http` 输入参数

- [来自 func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc)](https://github.com/kataras/iris/tree/master/_examples/convert-handlers/negroni-like/main.go)
- [来自 http.Handler or http.HandlerFunc](https://github.com/kataras/iris/tree/master/_examplesconvert-handlers/nethttp/main.go)
- [来自 func(http.HandlerFunc) http.HandlerFunc](https://github.com/kataras/iris/tree/master/_examplesconvert-handlers/real-usecase-raven/writing-middleware/main.go)

---

以下是一些专门为Iris制作的处理器列表:

## 内置的

| 中间件 | 例子 |
| --- | --- |
| [basic authentication](https://github.com/kataras/iris/tree/master/middleware/basicauth) | [iris/_examples/auth/basicauth](https://github.com/kataras/iris/tree/master/_examples/auth/basicauth) |
| [request logger](https://github.com/kataras/iris/tree/master/middleware/logger) | [iris/_examples/logging/request-logger](https://github.com/kataras/iris/tree/master/_examples/logging/request-logger) |
| [HTTP method override](https://github.com/kataras/iris/tree/master/middleware/methodoverride) | [iris/middleware/methodoverride/methodoverride_test.go](https://github.com/kataras/iris/blob/master/middleware/methodoverride/methodoverride_test.go) |
| [profiling (pprof)](https://github.com/kataras/iris/tree/master/middleware/pprof) | [iris/_examples/pprof](https://github.com/kataras/iris/tree/master/_examples/pprof) |
| [Google reCAPTCHA](https://github.com/kataras/iris/tree/master/middleware/recaptcha) | [iris/_examples/auth/recaptcha](https://github.com/kataras/iris/tree/master/_examples/auth/recaptcha) |
| [hCaptcha](https://github.com/kataras/iris/tree/master/middleware/hcaptcha) | [iris/_examples/auth/recaptcha](https://github.com/kataras/iris/tree/master/_examples/auth/hcaptcha) |
| [recovery](https://github.com/kataras/iris/tree/master/middleware/recover) | [iris/_examples/recover](https://github.com/kataras/iris/tree/master/_examples/recover) |
| [rate](https://github.com/kataras/iris/tree/master/middleware/rate) | [iris/_examples/request-ratelimit](https://github.com/kataras/iris/tree/master/_examples/request-ratelimit) |
| [jwt](https://github.com/kataras/iris/tree/master/middleware/jwt) | [iris/_examples/auth/jwt](https://github.com/kataras/iris/tree/master/_examples/auth/jwt) |
| [requestid](https://github.com/kataras/iris/tree/master/middleware/requestid) | [iris/middleware/requestid/requestid_test.go](https://github.com/kataras/iris/blob/master/_examples/middleware/requestid/requestid_test.go) |

## 社区

| 中间件 | 描述 | 例子 |
| --- | --- | --- |
| [jwt](https://github.com/iris-contrib/middleware/tree/master/jwt) | 中间件在传入请求的 `Authorization` 头上检查JWT, 并对其进行解码 | [iris-contrib/middleware/jwt/_example](https://github.com/iris-contrib/middleware/tree/master/jwt/_example) |
| [cors](https://github.com/iris-contrib/middleware/tree/master/cors) | HTTP访问控制 | [iris-contrib/middleware/cors/_example](https://github.com/iris-contrib/middleware/tree/master/cors/_example) |
| [secure](https://github.com/iris-contrib/middleware/tree/master/secure) | 实现一些快速安全性的中间件 | [iris-contrib/middleware/secure/_example](https://github.com/iris-contrib/middleware/tree/master/secure/_example/main.go) |
| [tollbooth](https://github.com/iris-contrib/middleware/tree/master/tollboothic) | 用于限制HTTP请求速率的通用中间件 | [iris-contrib/middleware/tollboothic/_examples/limit-handler](https://github.com/iris-contrib/middleware/tree/master/tollboothic/_examples/limit-handler) |
| [cloudwatch](https://github.com/iris-contrib/middleware/tree/master/cloudwatch) | AWS cloudwatch度量中间件 | [iris-contrib/middleware/cloudwatch/_example](https://github.com/iris-contrib/middleware/tree/master/cloudwatch/_example) |
| [new relic](https://github.com/iris-contrib/middleware/tree/master/newrelic) | 官方的 [New Relic Go Agent](https://github.com/newrelic/go-agent) |  [iris-contrib/middleware/newrelic/_example](https://github.com/iris-contrib/middleware/tree/master/newrelic/_example) |
| [prometheus](https://github.com/iris-contrib/middleware/tree/master/prometheus) | 为prometheus工具轻松地创建度量端点 | [iris-contrib/middleware/prometheus/_example](https://github.com/iris-contrib/middleware/tree/master/prometheus/_example) |
| [casbin](https://github.com/iris-contrib/middleware/tree/master/casbin) | 支持访问控制模型(如ACL、RBAC、ABAC)的授权库 | [iris-contrib/middleware/casbin/_examples](https://github.com/iris-contrib/middleware/tree/master/casbin/_examples) |
| [raven](https://github.com/iris-contrib/middleware/tree/master/raven) | Go的哨兵客户端 | [iris-contrib/middleware/raven/_example](https://github.com/iris-contrib/middleware/blob/master/raven/_example/main.go) |
| [csrf](https://github.com/iris-contrib/middleware/tree/master/csrf) | 跨站点请求伪造保护 | [iris-contrib/middleware/csrf/_example](https://github.com/iris-contrib/middleware/blob/master/csrf/_example/main.go) |
| [go-i18n](https://github.com/iris-contrib/middleware/tree/master/go-i18n) | i18n Iris 装载为 nicksnyder/go-i18n | [iris-contrib/middleware/go-i18n/_example](https://github.com/iris-contrib/middleware/blob/master/go-i18n/_example/main.go) |
| [throttler](https://github.com/iris-contrib/middleware/tree/master/throttler) | 对HTTP端点的访问速率限制 | [iris-contrib/middleware/throttler/_example](https://github.com/iris-contrib/middleware/blob/master/throttler/_example/main.go) |

## 第三方Party

Iris有自己的中间件形式的 `func(ctx Iris . context)`, 但它也兼容所有 `net/http` 中间件形式。看[这儿](https://github.com/kataras/iris/tree/master/_examples/convert-handlers)