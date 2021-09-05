# 路由覆盖上下文(Routing override context)

在本节中, 您将学习如何覆盖现有 [上下文](https://godoc.org/github.com/kataras/iris/context#Context)的方法

上下文是一个接口。然而, 你可能知道, 当您使用其他框架时, 您没有覆盖的功能, 即使它被用作一个接口。使用Iris, 您可以使用 `app.ContextPool.Attach` 方法来实现**附加**到**上下文池**本身

1.让我们从导入需要的 `github.com/kataras/iris/v12/context` 开始
2.其次，创建自己的上下文实现
3.添加 `Do` 和 `Next` 方法，它们只调用 `Context.Do` 和 `Context.Next` 包级的功能
4.使用应用的 `ContextPool` , 将其设置为应该用于路由处理程序的上下文实现

实例代码:

请阅读*评论*

```go
package main

import (
    "reflect"

    "github.com/kataras/iris/v12"
    // 1.
    "github.com/kataras/iris/v12/context"
)

// 2.
// 创建自己的自定义上下文，放入所需的任何字段
type MyContext struct {
    // 嵌入 `iris.Context` - 
    // 它完全是可选的，但如果你不想覆盖所有上下文的方法，你就需要这个！
    iris.Context
}

// 可选:验证 MyContext 在编译时实现了 iris.Context
var _ iris.Context = &MyContext{}

// 3.
func (ctx *MyContext) Do(handlers context.Handlers) {
    context.Do(ctx, handlers)
}


// 3.
func (ctx *MyContext) Next() {
    context.Next(ctx)
}

// [在这里重写任何你想要的上下文方法...]
// 就像下面的 HTML:

func (ctx *MyContext) HTML(format string, args ...interface{}) (int, error) {
    ctx.Application().Logger().Infof("Executing .HTML function from MyContext")

    ctx.ContentType("text/html")
    return ctx.Writef(format, args...)
}

func main() {
    app := iris.New()

    // 4.
    app.ContextPool.Attach(func() iris.Context {
        return &MyContext{
            // 如果你使用内嵌上下文，调用 `Context.NewContext` 创建一个:
            Context: context.NewContext(app),
        }
    })

    // 在./view/**目录下的.html文件上注册一个视图引擎
    app.RegisterView(iris.HTML("./view", ".html"))

    // 像往常一样注册你的路由
    app.Handle("GET", "/", recordWhichContextForExample,
    func(ctx iris.Context) {
        // 使用上下文覆盖的HTML方法
        ctx.HTML("<h1> Hello from my custom context's HTML! </h1>")
    })

    // 当MyContext没有直接定义自己的视图函数时
    // 这将由嵌入默认上下文 MyContext.Context 执行
    app.Handle("GET", "/hi/{firstname:alphabetical}",recordWhichContextForExample,
    func(ctx iris.Context) {
        firstname := ctx.Values().GetString("firstname")

        ctx.ViewData("firstname", firstname)
        ctx.Gzip(true)

        ctx.View("hi.html")
    })

    app.Listen(":8080")
}

// 应该总是打印 "($PATH)处理程序正在从'MyContext'执行"
func recordWhichContextForExample(ctx iris.Context) {
    ctx.Application().Logger().Infof("(%s) Handler is executing from: '%s'",
        ctx.Path(), reflect.TypeOf(ctx).Elem().Name())

    ctx.Next()
}
```
