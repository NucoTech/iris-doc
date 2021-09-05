# 路由错误处理器(Routing error handlers)

## 错误处理器

当特定的http错误代码发生时, 您可以定义自己的处理程序; 每一方都可以注册错误处理程序。

错误码是大于或等于400的http状态码, 如 404 未找到 和 500 内部服务器错误。

实例代码:

```go
package main

import "github.com/kataras/iris/v12"

func main(){
    app := iris.New()
    app.RegisterView(iris.HTML("./views", ".html"))

    app.OnErrorCode(iris.StatusNotFound, notFound)
    app.OnErrorCode(iris.StatusInternalServerError, internalServerError)
    // 为所有错误代码注册处理程序:
    // app.OnAnyErrorCode(处理器)

    app.Get("/", index)

    app.Listen(":8080")
}

func notFound(ctx iris.Context) {
    // 当404页面时呈现模板
    // $views_dir/errors/404.html
    ctx.View("errors/404.html")

    // 或者, 如果你有更多的路径并且你想向用户推荐相对路径:
    // suggestPaths := ctx.FindClosest(3)
    // if len(suggestPaths) == 0 {
    //     ctx.WriteString("404 not found")
    //     return
    // }

    // ctx.HTML("Did you mean?<ul>")
    // for _, s := range suggestPaths {
    //     ctx.HTML(`<li><a href="%s">%s</a></li>`, s, s)
    // }
    // ctx.HTML("</ul>")
}

func internalServerError(ctx iris.Context) {
    ctx.WriteString("Oups something went wrong, try again")
}

func index(ctx iris.Context) {
    ctx.View("index.html")
}
```

> 在[View](/View.md)中学习更多

Iris有 `EnablePathIntelligence` 选项, 可以传递给 `app.Run/Listen` 方法, 自动重定向一个未找到的路径到最匹配的路径, 例如 `"http://localhost:8080/contac"` 到 `"http://localhost:8080/contact"` 启用它:

```go
app.Listen(":8080", iris.WithPathIntelligence)
```

您还可以通过设置 `ForceLowercaseRouting` 选项强制所有路径小写:

```go
app.Listen(":8080", iris.WithLowercaseRouting, iris.WithPathIntelligence)
```

## 问题类型

Iris为 [Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807) 提供了内置支持

`Context.Problem` 写一个JSON或XML问题响应。行为和 `Context.JSON` 完全一样, 但带有默认的 ProblemOptions.JSON缩进为" ", 响应内容类型为 "application/problem+json"

使用这个 option.RenderXML 和 XML 字段来改变这种行为, 并发送一个内容类型为 "application/problem+XML" 的响应

```go
func newProductProblem(productName, detail string) iris.Problem {
    return iris.NewProblem().
        // 类型URI, 如果是相对的, 它会自动转换为绝对的
        Type("/product-error"). 
        // 标题, 如果为空, 则从状态码获取它
        Title("Product validation problem").
        // 任何可选的细节
        Detail(detail).
        // 状态错误码, 必需
        Status(iris.StatusBadRequest).
        // 任何自定义键-值对
        Key("productName", productName)
        // 可选的问题原因, 连锁的问题
        // .Cause(other iris.Problem)
}

func fireProblem(ctx iris.Context) {
    // 响应像JSON但与缩进的" "和内容类型 "application/problem+json"
    ctx.Problem(newProductProblem("product name", "problem details"), iris.ProblemOptions{
        // 可选的JSON渲染器设置
        JSON: iris.JSON{
            Indent: "  ",
        },
        // 或者
        // 呈现为XML:
        // RenderXML: true,
        // XML:       iris.XML{Indent: "  "},
        // 设置 "Retry-After" 响应头
        //
        // 可以接收:
        // time.Time for HTTP-Date,
        // time.Duration, int64, float64, int for seconds
        // 或字符串表示日期或持续时间
        // 例子:
        // time.Now().Add(5 * time.Minute),
        // 300 * time.Second,
        // "5m",
        //
        RetryAfter: 300,
        // 一个函数, 如果指定了, 可以根据请求动态设置重试后重试
        // 对于 ProblemOptions 的可重用性很有用。

        //重写RetryAfter字段
        //
        // RetryAfterFunc: func(iris.Context) interface{} { [...] }
    })
}
```

## 输出"application/problem+json"

```json
{
  "type": "https://host.domain/product-error",
  "status": 400,
  "title": "Product validation problem",
  "detail": "problem error details",
  "productName": "product name"
}
```

当 `RenderXML` 设置为 `true` 时, 响应将看起来是作为 XML 呈现的

## 输出"application/problem+xml"

```XML
<Problem>
    <Type>https://host.domain/product-error</Type>
    <Status>400</Status>
    <Title>Product validation problem</Title>
    <Detail>problem error details</Detail>
    <ProductName>product name</ProductName>
</Problem>
```

完整的例子请访问[_examples/routing/http-errors](https://github.com/kataras/iris/blob/master/_examples/routing/http-errors/main.go)
