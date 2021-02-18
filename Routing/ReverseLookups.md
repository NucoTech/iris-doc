# 逆向查找(Routing reverse lookups)

## 逆向查找

正如 [路由](Routing\README.md) 章节中提到的, Iris提供了几个处理程序注册方法, 每个方法都返回一个 [路由](https://godoc.org/github.com/kataras/iris/core/router#Route) 实例

## 路线命名

路由命名很简单, 因为我们只需使用 `Name` 字段调用返回的 `*Route` 来定义名称:

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()
    // 定义一个函数
    h := func(ctx iris.Context) {
        ctx.HTML("<b>Hi</b1>")
    }

    // 处理程序注册和命名
    home := app.Get("/", h)
    home.Name = "home"
    // 或者
    app.Get("/about", h).Name = "about"
    app.Get("/page/{id}", h).Name = "page"

    app.Listen(":8080")
}
```

## 反向路由又名从路由名生成url

当我们为特定路径注册处理程序时, 我们能够基于传递给Iris的结构化数据创建url。在上面的例子中, 我们已经命名了三个路由器, 其中一个甚至接受参数。如果我们使用默认的 `html/template` 视图引擎, 我们可以使用一个简单的动作来反向路由(并生成实际的url):

```xml
Home: {{ urlpath "home" }}
About: {{ urlpath "about" }}
Page 17: {{ urlpath "page" "17" }}
```

```xml
Home: http://localhost:8080/ 
About: http://localhost:8080/about
Page 17: http://localhost:8080/page/17
```

## 在代码中使用路由名

我们可以使用以下方法/函数来处理命名路由(及其参数):

- [GetRoutes](https://godoc.org/github.com/kataras/iris/core/router#APIBuilder.GetRoutes) 函数获取所有注册的路由
- [GetRoute(routeName string)](https://godoc.org/github.com/kataras/iris/core/router#APIBuilder.GetRoute) 方法按名称检索路由
- [URL(routeName string, paramValues ...interface{})](https://godoc.org/github.com/kataras/iris/core/router#RoutePathReverser.URL) 方法根据提供的参数生成url字符串
- [Path(routeName string, paramValues ...interface{}](https://godoc.org/github.com/kataras/iris/core/router#RoutePathReverser.Path) 方法根据提供的值仅生成URL的路径(不包含主机和协议)部分
