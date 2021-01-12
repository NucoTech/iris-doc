# 视图层(View)

Iris通用的[视图引擎](https://godoc.org/github.com/kataras/iris/view#Engine)提供了6个即用的模型解析器。 当然, 开发人员依然可以使用各种go模板解析器作为 `Context.ResponseWriter()` 来完成 `http.ResponseWriter` 和 `io.Writer`

Iris添加了一些通用的规则和特性, 他们的原生解析器并不支持这些, 例如, 我们支持 `yield` , `render` , `render_r` , `current` , `urlpath` 模板功能, 并且 `Layouts` 和 `binding` 跨越引擎所有的中间件和**嵌入式模板文件**

要使用模板引擎的独特特性, 你需要阅读文档来学习特性和语法(点击下方链接), 选择您的程序所需要的

以下是已经内置的视图引擎列表

| 引擎 | 声明 | 标注模板解析器 |
| --- | --- | --- |
| std template/html | `iris.HTML(...)` | [html/template](https://golang.org/pkg/html/template/) package |
| django | `iris.Django(...)` | [iris-contrib/pongo2](https://github.com/iris-contrib/pongo2) package |
| handlebars | `iris.Handlebars(...)` | [Joker/jade](https://github.com/Joker/jade) package |
| amber | `iris.Amber(...)` | [aymerick/raymond](https://github.com/aymerick/raymond) package |
| pug(jade) | `iris.Pug(...)` | [eknkc/amber](https://github.com/eknkc/amber) package |
| jet | `iris.Jet(...)` | [CloudyKit/jet](https://github.com/CloudyKit/jet) package |

一个或多个视图引擎可以在同一个应用中注册, 我们需要用 `RegisterView(ViewEngine)` 方法**注册**一个视图引擎

从 "./views" 文件夹中加载所有扩展名为 ".html" 的模板, 然后用标准 `html/template` 模板包来解析他们

```go
// [app := iris.New...]
tmpl := iris.HTML("./views", ".html")
app.RegisterView(tmpl)
```

用 `Context.View` 方法来**呈现或执行**一个内嵌在主路由处理程序中的视图层

```go
ctx.View("hi.html")
```

要通过中间件或主处理程序来**绑定**内嵌到视图层的go键值对模式的值, 在 `Coontext.View` 使用 `Context.ViewData` 方法

绑定: `{{.message}} with "Hello world!"`

```go
ctx.ViewData("message", "Hello world!")
```

为了绑定视图层中go的**模型**, 你需要注意两点

- `ctx.ViewData("user", User{})` - 变量绑定在 `{{.user.Name}}` 为例
- `ctx.View("user-page.html", User{})` - 根绑定 `{{.Name}}` 为例

要**添加模板函数**, 请选择优先视图引擎的 `AddFunc` 方法

```go
// 函数名, 输入函数, 渲染值
tmpl.AddFunc("greet", func(s string) string {
    return "Greetings " + s + "!"
})
```

**本地文件更改后重新加载**, 可调用视图引擎的 `Reload` 方法

```go
tmpl.Reload(true)
```

要使用**嵌入式**模板而不是依赖本地的文件系统, 请使用外部工具: [go-bindata](https://github.com/go-bindata/go-bindata) 将 `Assent` 和 `AssetNames` 函数发送给优先视图引擎的 `Binary` 方法

```go
tmpl.Binary(Asset, AssetNames)
```

代码示例:

```go
// file: main.go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    // 从 "./views" 文件夹解析所有模板
    // 解析所有扩展名为 ".html" 的文件
    //使用 `html/template` 标准包
    tmpl := iris.HTML("./views", ".html")

    // 加载在本地模板文件上重新生成的更改文件
    tmpl.Reload(true)

    // 默认模板函数为:
    //
    // - {{ urlpath "myNamedRoute" "pathParameter_ifNeeded" }}
    // - {{ render "header.html" }}
    // and partial relative path to current page:
    // - {{ render_r "header.html" }} 
    // - {{ yield }}
    // - {{ current }}
    // Register a custom template func:
    tmpl.AddFunc("greet", func(s string) string {
        return "Greetings " + s + "!"
    })

    // 在视图层注册视图引擎,
    // 该行会加载模板
    app.RegisterView(tmpl)

    // 方法:    GET
    // 资源地址:  http://localhost:8080
    app.Get("/", func(ctx iris.Context) {
        // 绑定: {{.message}} with "Hello world!"
        ctx.ViewData("message", "Hello world!")
        // 渲染模板文件: ./views/hi.html
        ctx.View("hi.html")
    })

    app.Listen(":8080")
}
```

```html
<!-- file: ./views/hi.html -->
<html>
<head>
    <title>Hi Page</title>
</head>
<body>
    <h1>{{.message}}</h1>
    <strong>{{greet "to you"}}</strong>
</body>
</html>
```

浏览器搜索: http://localhost:8080

**渲染结果**看起来像:

```html
<html>
<head>
    <title>Hi Page</title>
</head>
<body>
    <h1>Hello world!</h1>
    <strong>Greetings to you!</strong>
</body>
</html>
```
