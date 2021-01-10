# Quick start

新建一个空的文件, 命名为`example.go`, 打开它并将下列代码粘贴进去

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.Default()
    app.Use(myMiddleware)
    app.Handle("GET", "/ping", func(ctx iris.Context) {
        ctx.JSON(iris.Map{"message": "pong"})
    })
    app.Listen(":8080")
}

func myMiddleware(ctx iris.Context) {
    ctx.Application().Logger().Infof("Runs before %s", ctx.Path())
    ctx.Next()
}
```

打开一个终端，进入`example.go`所在的目录，粘贴并执行下列代码

```bash
#运行 example.go 在浏览器访问http://localhost:8080/ping
go run example.go
```

## ShowMeMore

用一个简单的概述看一下iris上手有多么的简单

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()
    // 从 "./views" 目录加载所有文件后缀名是".html"的模板
    // 使用标准的 'html/template' 包解析它们
    app.RegisterView(iris.HTML("./views", ".html"))
    // Method:    GET
    // Resource:  http://localhost:8080
    app.Get("/", func(ctx iris.Context) {
        // Bind: {{.message}} with "Hello world!"
        ctx.ViewData("message", "Hello world!")
        // Render template file: ./views/hello.html
        ctx.View("hello.html")
    })
    // Method:    GET
    // Resource:  http://localhost:8080/user/42
    //
    // 需要使用一个自定义的正则表达式解析地址中的参数?
    // 很容易的;
    // 只需要将参数的类型标记为 'string'
    // 这样它就能使用宏定义的正则表达式来接受任意参数, 例如:
    // app.Get("/user/{id:string regexp(^[0-9]+$)}")
    app.Get("/user/{id:uint64}", func(ctx iris.Context) {
        userID, _ := ctx.Params().GetUint64("id")
        ctx.Writef("User ID: %d", userID)
    })
    // Start the server using a network address.
    app.Listen(":8080")
}
```

```html
<!-- file: ./views/hello.html -->
<html>
<head>
    <title>Hello Page</title>
</head>
<body>
    <h1>{{.message}}</h1>
</body>
</html>
```

> 想让你的项目在代码修改之后自动重新运行吗?安装[iris-cli](https://github.com/kataras/iris-cli)工具, 使用`iris-cli run`而不是`go run main.go`

在以下部分我们会了解更多关于`Routing`的内容
