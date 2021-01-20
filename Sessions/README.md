# 会话(Sessions)

使用应用程序时, 您打开它, 做一些更改, 然后关闭它, 这很像一个会话。电脑知道你是谁, 它知道什么时候启动应用, 什么时候结束应用。 但是在互联网上有一个问题: web服务器不知道您是谁、您在做什么, 因为HTTP地址不维护状态

会话变量通过存储跨多个页面使用的用户信息(例如用户名、最喜欢的颜色等)来解决这个问题。默认情况下, 会话变量一直持续到用户关闭浏览器。

所以, 会话变量保存单个用户的信息, 可以用于一个应用中的所有页面

> 小贴士: 如果您需要永久储存, 您可能想要存到[数据库](Sessions/Database.md)中

Iris有自己的会话实现, 会话管理器位于[iris/sessions](https://github.com/kataras/iris/tree/master/sessions)子包中, 您需要导入这个包才能使用HTTP会话

会话通过使用新的包级函数 `Sessions` 对象的 `Start` 函数来启动, 该函数将返回一个 `Session`

会话变量和 `Session.Set` 方法一起设置, 并且通过 `Session.Get` 及相关方法检索。想要删除单个变量需要使用 `Session.Delete`,而删除或取消整个会话则需要 `Session.Destroy` 方法

<details>
    <summary>来看看怎么写</summary>

会话管理器是使用 `New` 包级函数创建的

```go
    import "github.com/kataras/iris/v12/sessions"
    sess := sessions.New(sessions.Config{Cookie: "cookieName", ...})
```

`Config` 看起来像这样

```go
    SessionIDGenerator func(iris.Context) string

    // 默认为 "irissessionid".
    Cookie string

    CookieSecureTLS bool

    // 默认为 false.
    AllowReclaim bool

    // 默认为 nil.
    Encode func(cookieName string, value interface{}) (string, error)
    // 默认为 nil.
    Decode func(cookieName string, cookieValue string, v interface{}) error
    // 默认为 nil.
    Encoding Encoding

    // 默认为 infinitive/unlimited life duration(0)
    Expires time.Duration

    // 默认为 false
    DisableSubdomainPersistence bool
```

返回值是一个会话指针, 导出以下的方法

```go
    Start(ctx iris.Context,
    cookieOptions ...iris.CookieOption) *Session

    Handler(cookieOptions ...iris.CookieOption) iris.Handler

    Destroy()
    DestroyAll()
    DestroyByID(sessID string)
    OnDestroy(callback func(sid string))

    ShiftExpiration(ctx iris.Context,
        cookieOptions ...iris.CookieOption) error
    UpdateExpiration(ctx iris.Context, expires time.Duration,
        cookieOptions ...iris.CookieOption) error

    UseDatabase(db Database)
```

其中 `CookieOption` 只是一个 `func(*http.Cookie)` , 它允许自定义cookie的属性

`Start` 方法返回一个会话指针值, 该值导出自己的方法用于每个会话

```go
    func (ctx iris.Context) {
        session := sess.Start(ctx)
        .ID() string
        .IsNew() bool

        .Set(key string, value interface{})
        .SetImmutable(key string, value interface{})
        .GetAll() map[string]interface{}
        .Len() int
        .Delete(key string) bool
        .Clear()

        .Get(key string) interface{}
        .GetString(key string) string
        .GetStringDefault(key string, defaultValue string) string
        .GetInt(key string) (int, error)
        .GetIntDefault(key string, defaultValue int) int
        .Increment(key string, n int) (newValue int)
        .Decrement(key string, n int) (newValue int)
        .GetInt64(key string) (int64, error)
        .GetInt64Default(key string, defaultValue int64) int64
        .GetFloat32(key string) (float32, error)
        .GetFloat32Default(key string, defaultValue float32) float32
        .GetFloat64(key string) (float64, error)
        .GetFloat64Default(key string, defaultValue float64) float64
        .GetBoolean(key string) (bool, error)
        .GetBooleanDefault(key string, defaultValue bool) bool

        .SetFlash(key string, value interface{})
        .HasFlash() bool
        .GetFlashes() map[string]interface{}

        .PeekFlash(key string) interface{}
        .GetFlash(key string) interface{}
        .GetFlashString(key string) string
        .GetFlashStringDefault(key string, defaultValue string) string

        .DeleteFlash(key string)
        .ClearFlashes()

        .Destroy()
    }
```

</details>

## 例子

在本例中, 我们只允许通过身份验证过的用户在/secret 页面上查看我们的隐藏消息。想拥有权限, 首先必须访问 /login 以获得有效的会话cookie。此外, 他可以访问 /logout 注销他访问我们隐藏消息的权限

```go
// sessions.go
package main

import (
    "github.com/kataras/iris/v12"

    "github.com/kataras/iris/v12/sessions"
)

var (
    cookieNameForSessionID = "mycookiesessionnameid"
    sess                   = sessions.New(sessions.Config{Cookie: cookieNameForSessionID})
)

func secret(ctx iris.Context) {
    // 检查用户是否已通过身份验证
    if auth, _ := sess.Start(ctx).GetBoolean("authenticated"); !auth {
        ctx.StatusCode(iris.StatusForbidden)
        return
    }

    // 打印隐藏信息
    ctx.WriteString("The cake is a lie!")
}

func login(ctx iris.Context) {
    session := sess.Start(ctx)

    // 通过身份验证会跳转到这儿
    // ...

    // 将用户设置为 "已通过验证的"
    session.Set("authenticated", true)
}

func logout(ctx iris.Context) {
    session := sess.Start(ctx)

    // 撤掉用户的验证
    session.Set("authenticated", false)
    // 或者删掉变量:
    session.Delete("authenticated")
    // 又或者干掉整个会话:
    session.Destroy()
}

func main() {
    app := iris.New()

    app.Get("/secret", secret)
    app.Get("/login", login)
    app.Get("/logout", logout)

    app.Listen(":8080")
}
```

```shell
go run sessions.go

curl -s http://localhost:8080/secret
Forbidden

curl -s -I http://localhost:8080/login
Set-Cookie: mysessionid=MTQ4NzE5Mz...

curl -s --cookie "mysessionid=MTQ4NzE5Mz..." http://localhost:8080/secret
The cake is a lie!
```

## 中间件(Midddleware)

当您希望在相同的请求处理程序和生命周期(处理程序链、中间件)中使用 `Session` , 您可以选择将其注册为一个中间件并使用包级会话 `Session.Get` 来检索存储到上下文的会话

`Session` 结构体的值包含`Handler` 方法,该方法可用于返回将被注册为中间件的 `iris.Handler`

```go
import "github.com/kataras/iris/v12/sessions"

sess := sessions.New(sessions.Config{...})

app := iris.New()
app.Use(sess.Handler())

app.Get("/path", func(ctx iris.Context){
    session := sessions.Get(ctx)
    // [使用会话...]
})
```

同样的, 如果会话管理器的 `Config.AllowReclaim` 置于 `true` , 在同一个生命周期中您仍可调用 `sess.Start` ,想用多少次就用多少次, 并且不需要注册为中间件

[点我](https://github.com/kataras/iris/tree/master/_examples/sessions)查看sessions子包的更多信息

继续查看[Flash Message](Sessions/FlashMessages.md)章节
