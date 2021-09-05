# 包装路由 (Wrap the Router)

您可能永远不需要这个, 但以防万一

有时, 您可能需要覆盖或决定路由器是否将在传入请求上执行。如果你以前有过使用 `net/http` 和其他web框架的经验, 你会对这个函数很熟悉(它有net/http中间件的形式, 但它不接受下一个处理程序, 而是接受路由器作为一个函数来执行或不执行)

```go
// WrapperFunc用作WrapRouter的预期输入参数签名。它是一种 "低级" 签名, 与net/http兼容
// 它被用于运行或不运行基于自定义逻辑的路由器
type WrapperFunc func(w http.ResponseWriter, r *http.Request, router http.HandlerFunc)

// WrapRouter在主路由器的顶部添加一个包装器
// 通常, 当需要用CORS这样的中间件包装整个应用程序时, 它对第三方中间件很有用

// Developers可以添加多个包装器, 这些包装器的执行由小到大。
// 这意味着第二个包装器将包装第一个, 以此类推

// 在构建之前
func WrapRouter(wrapperFunc WrapperFunc)
```

路由器根据 `Subdomain` 、`HTTP Method` 和动态 `Path` 查找路由。路由器包装器可以覆盖该行为并执行自定义代码。

在本例中, 您将看到 .WrapRouter的一个用例。为了执行注册的路由处理程序, 你可以使用 .WrapRouter来添加自定义逻辑。这只是为了证明概念, 您可以跳过本教程

实例代码:

```go
package main

import (
    "net/http"
    "strings"

    "github.com/kataras/iris/v12"
)

func newApp() *iris.Application {
    app := iris.New()

    app.OnErrorCode(iris.StatusNotFound, func(ctx iris.Context) {
        ctx.HTML("<b>Resource Not found</b>")
    })

    app.HandleDir("/", "./public")

    app.Get("/profile/{username}", func(ctx iris.Context) {
        ctx.Writef("Hello %s", ctx.Params().Get("username"))
    })

    myOtherHandler := func(ctx iris.Context) {
        ctx.Writef("inside a handler which is fired manually by our custom router wrapper")
    }
    // 用本机net/http处理程序包装路由器
    // 如果url不包含任何 "."(即:. css, js…)
    // 根据应用程序, 你可能需要添加更多的文件服务器异常
    // 处理程序将执行负责的路由器
    // 注册路由(查看 "/" 和 "/profile/{username}")
    // 如果不是, 它将根据根目录 "/" 提供文件
    app.WrapRouter(func(w http.ResponseWriter, r *http.Request, router http.HandlerFunc) {
        path := r.URL.Path

        if strings.HasPrefix(path, "/other") {
                // 获取并释放一个上下文, 以便使用它来执行我们的自定义处理程序

                // 记住:如果你要调用一个net/http中间件, 那么你不需要在这里获取和释放上下文
                ctx := app.ContextPool.Acquire(w, r)
                myOtherHandler(ctx)
                app.ContextPool.Release(ctx)
                return
            }

            // 否则, 继续正常的服务路由
            router.ServeHTTP(w, r) 
    })

    return app
}

func main() {
    app := newApp()

    // http://localhost:8080
    // http://localhost:8080/index.html
    // http://localhost:8080/app.js
    // http://localhost:8080/css/main.css
    // http://localhost:8080/profile/anyusername
    // http://localhost:8080/other/random
    app.Listen(":8080")

    // 注意:在这个例子中, 我们只看到了一个用例
    // 你可能想要通过 .WrapRouter或 .Downgrade来绕过Iris的默认路由器, 即:
    // 你也可以使用这个方法来设置自定义代理  
}
```

这里没有太多要说的, 它只是一个函数包装器, 它接受本地响应编写器和请求, 下一个处理程序是Iris路由器本身, 它是否被执行, 是否被调用, **它是整个路由器的中间件**
