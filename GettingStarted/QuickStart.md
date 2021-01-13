# 快速开始(QuickStart)

新建一个空文件, 命名为`example.go`, 打开它并复制粘贴下面的代码。

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.Default()
    app.Use(myMiddleware)
    app.Handle("GET", "/ping", func(ctx iris.Context) {
        ctx.JSON(iris.Map{"message": "pong"})
    })
    // 监听服务入向http请求 http://localhost:8080
    app.Listen(":8080")
}

func myMiddleware(ctx iris.Context) {
    ctx.Application().Logger().Infof("Runs before %s", ctx.Path())
    ctx.Next()
}
```

打开一个终端会话，进入`example.go`所在的目录，执行下面的内容。

```shell
# 运行 example.go 在浏览器访问http://localhost:8080/ping
go run example.go
```

## 学到更多!(ShowMeMore)

简单概述一下上手有多简单

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
    // 使用一个自定义的正则表达式来替代?
    // 只需要将参数的类型标记为字符串
    // 这样就能使用它的宏函数`regexp`来接受任何的参数, 
    // 例如: app.Get("/user/{id:string regexp(^[0-9]+$)}")
    app.Get("/user/{id:uint64}", func(ctx iris.Context) {
        userID, _ := ctx.Params().GetUint64("id")
        ctx.Writef("User ID: %d", userID)
    })
    // 使用网络地址启动服务
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

> 想让你的项目在代码修改之后自动重新运行吗? 安装[iris-cli](https://github.com/kataras/iris-cli)工具, 并用`iris-cli run`替代`go run main.go`执行。

在下面的章节我们将学到更多关于[路由(Routing)](https://github.com/kataras/iris/wiki/Routing)
