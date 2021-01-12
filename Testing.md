# 测试

Iris为[httpexpect](https://github.com/gavv/httpexpect)提供了一个令人难以置信的支持, 一个web应用程序的测试框架. `iris/httptest`子包为Iris + httpexpect提供帮助

如果你更倾向于使用Go标准的[net/http/httptest](https://golang.org/pkg/net/http/httptest/)包, 你仍可以使用它. Iris如每一个http web框架一样, 可以兼容一些外部的测试工具, 最终还是HTTP

## 基础验证

在我们的第一个例子, 我们将使用`iris/httptest`来测试基础验证

**1.** `main.go`的源文件长这样

```go
package main

import (
    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/middleware/basicauth"
)

func newApp() *iris.Application {
    app := iris.New()

    authConfig := basicauth.Config {
        Users: map[string]string{"myusername": "mypassword"},
    }

    authentication := basicauth.New(authConfig)

    app.Get("/", func(ctx iris.Context) { ctx.Redirect("/admin") })

    needAuth := app.Party("/admin", authentication)
    {
        // http://localhost:8080/admin
        needAuth.Get("/", h)
        // http://localhost:8080/admin/profile
        needAuth.Get("/profile", h)

        // http://localhost:8080/admin/settings
        needAuth.Get("/settings", h)
    }

    return app
}

func h(ctx iris.Context) {
    username, password, _ := ctx.Request().BasicAuth()
    // 第三个参数 ^ 因为中间件的原因为true
    // 谨记这些, 否则这个处理不会被执行

    ctx.Writef("%s %s %s", ctx.Path(), username, password)
}

func main() {
    app := newApp()
    app.Listen(":8080")
}
```

**2.** 现在创建一个`main_test`文件, 并且复制粘贴下面的内容

```go
package main

import (
    "testing"

    "github.com/kataras/iris/v12/httptest"
)

func TestNewApp(t *testing.T) {
    app := newApp()
    e := httptest.New(t, app)

    // 如果没有基础验证, 重定向到 /admin
    e.GET("/").Expect().Status(httptest.StatusUnauthorized)
    // 不带有基础验证
    e.GET("/admin").Expect().Status(httptest.StatusUnauthorized)

    // 带有合法的基础验证
    e.GET("/admin").WithBasicAuth("myusername", "mypassword").Expect().
        Status(httptest.StatusOK).Body().Equal("/admin myusername:mypassword")
    e.GET("/admin/profile").WithBasicAuth("myusername", "mypassword").Expect().
        Status(httptest.StatusOK).Body().Equal("/admin/profile myusername:mypassword")
    e.GET("/admin/settings").WithBasicAuth("myusername", "mypassword").Expect().
        Status(httptest.StatusOK).Body().Equal("/admin/settings myusername:mypassword")

    // 带有非法的基础验证
    e.GET("/admin/settings").WithBasicAuth("invalidusername", "invalidpassword").
        Expect().Status(httptest.StatusUnauthorized)
}
```

**3.** 打开你的命令行工具并执行

```shell
go test -v
```

## 另一个例子: cookies

```go
package main

import (
    "fmt"
    "testing"

    "github.com/kataras/iris/v12/httptest"
)

func TestCookiesBasic(t *testing.T) {
    app := newApp()
    e := httptest.New(t, app, httptest.URL("http://example.com"))

    cookieName, cookieValue := "my_cookie_name", "my_cookie_value"

    // 测试设置一个 Cookie
    t1 := e.GET(fmt.Sprintf("/cookies/%s/%s", cookieName, cookieValue)).
        Expect().Status(httptest.StatusOK)
    // 验证Cookie的存在性, 现在它应当是可用的
    t1.Cookie(cookieName).Value().Equal(cookieValue)
    t1.Body().Contains(cookieValue)

    path := fmt.Sprintf("/cookies/%s", cookieName)

    // 测试检索一个Cookie
    t2 := e.GET(path).Expect().Status(httptest.StatusOK)
    t2.Body().Equal(cookieValue)

    // 测试移除一个Cookie
    t3 := e.DELETE(path).Expect().Status(httptest.StatusOK)
    t3.Body().Contains(cookieName)

    t4 := e.GET(path).Expect().Status(httptest.StatusOK)
    t4.Cookies().Empty()
    t4.Body().Empty()
}
```

```shell
go test -v -run=TestCookiesBasic$
```

Iris web框架自己使用自己的包来测试自己。在 [_examples仓库目录](https://github.com/kataras/iris/tree/master/_examples) 你可以找到一些有用的测试。获取更多的信息, 请看一看 [httpexpect的文档](https://github.com/gavv/httpexpect)
