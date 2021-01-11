# API Versioning

[版本控制](https://github.com/kataras/iris/tree/master/versioning)子包为您的API提供[semver](https://semver.org/)版本控制, 它实现了在[API指南](https://github.com/byrondover/api-guidelines/blob/master/Guidelines.md#versioning)中写的所有建议

版本的比较由[go版本管理](https://github.com/hashicorp/go-version)包完成, 它支持在 ">= 1.0, < 3" 等模式上进行匹配

```go
import (
    // [...]

    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/versioning"
)
```

## 特性(Features)

- 对于每个路由版本的匹配, 一个带有 "switch" 事件的普通iris处理程序通过Map管理版本=> 处理程序
- 每组版本化的路由和弃用API
- 版本匹配如 ">= 1.0，< 2.0" 或只是 "2.0.1" 等
- 版本未找到处理程序(可以通过简单地添加版本控制来定制, 未找到: Map上的未匹配的自定义版本处理)
- 从 "Accept" 和 "Accept-Version" 头中检索版本(可以通过中间件定制)
- 如果找到了对应版本，则返回 "X-API-Version"头文件
- 弃用选项，通过弃用装饰器定制 "X-API-Warn" ,  "X-API-Deprecation-Date", "X-API-Deprecation-Info" 头文件

## 获取版本(Get version)

当前请求版本由 `versioning.GetVersion(ctx)` 检索

默认情况下, GetVersion将会尝试读入:

- `Accept header, i.e Accept: "application/json; version=1.0"`
- `Accept-Version header, i.e Accept-Version: "1.0"`

您还可以通过中间件, 利用上下文的存储值为处理程序设置自定义版本, 例如:

```go
func(ctx iris.Context) {
    ctx.Values().Set(versioning.Key, ctx.URLParamDefault("version", "1.0"))
    ctx.Next()
}
```

## 为处理程序匹配版本(Match version to handler)

`versioning.NewMatcher(versioning.Map) iris.Handler` 创建一个单独的处理程序, 该程序根据请求的版本执行处理程序

```go
app := iris.New()

// 所有版本的中间件
myMiddleware := func(ctx iris.Context) {
    // [...]
    ctx.Next()
}

myCustomNotVersionFound := func(ctx iris.Context) {
    ctx.StatusCode(iris.StatusNotFound)
    ctx.Writef("%s version not found", versioning.GetVersion(ctx))
}

userAPI := app.Party("/api/user")
userAPI.Get("/", myMiddleware, versioning.NewMatcher(versioning.Map{
    "1.0":               sendHandler(v10Response),
    ">= 2, < 3":         sendHandler(v2Response),
    versioning.NotFound: myCustomNotVersionFound,
}))
```

### 弃用(Deprecation)

使用 `versioning.Deprecated(handler iris.Handler, options versioning.DeprecationOptions) iris.Handler` 功能, 您可以将特定的处理程序版本标记为弃用

```go
v10Handler := versioning.Deprecated(sendHandler(v10Response), versioning.DeprecationOptions{
    // 如果默认为空: "警告! 您正在使用该API的一个弃用版本"
    WarnMessage string 
    DeprecationDate time.Time
    DeprecationInfo string
})

userAPI.Get("/", versioning.NewMatcher(versioning.Map{
    "1.0": v10Handler,
    // [...]
}))
```

处理程序将会将这些数据头发送给客户端:

- "X-API-Warn": options.WarnMessage
- "X-API-Deprecation-Date": context.FormatTime(ctx, options.DeprecationDate))
- "X-API-Deprecation-Info": options.DeprecationInfo

如果您不在意时间和信息,可以传递 `versioning.DefaultDeprecationOptions` 来代替

## 按版本对路由进行分组(Grouping routes by version)

也可以根据版本对路由进行分组

使用 `versioning.NewGroup(version string) *versioning.Group` 功能, 您可以创建一个组来注册版本化的路由, `versioning.RegisterGroups(r iris.Party, versionNotFoundHandler iris.Handler, groups ...*versioning.Group)` 必须在最后被调用, 为了把路由注册到一个特定的 `Party`

```go
app := iris.New()

userAPI := app.Party("/api/user")
// [... 静态服务,中间件等都写入这里].

userAPIV10 := versioning.NewGroup("1.0")
userAPIV10.Get("/", sendHandler(v10Response))

userAPIV2 := versioning.NewGroup(">= 2, < 3")
userAPIV2.Get("/", sendHandler(v2Response))
userAPIV2.Post("/", sendHandler(v2Response))
userAPIV2.Put("/other", sendHandler(v2Response))

versioning.RegisterGroups(userAPI, versioning.NotFoundHandler, userAPIV10, userAPIV2)
```

一个中间件只能注册到实际的 `iris.Party` , 使用上面学到的方法,即当 "x" 或没有请求版本时使用 `versioning.Match` 来检测您想要执行的代码/控制程序

### 组弃用(Deprecation for Group)

只需要对想要弃用的组使用 `Deprecated(versioning.DeprecationOptions)` , 来通知API使用者这个特定的版本已经被弃用

```go
userAPIV10 := versioning.NewGroup("1.0").Deprecated(versioning.DefaultDeprecationOptions)
```

## 在处理程序中手动比较版本(Compare version manually from inside your handlers)

```go
// 返回 "version" 是否与 "is" 的内容匹配.
// "is" 的内容可以被限定在 ">= 1, < 3".
If(version string, is string) bool
```

```go
app.Get("/api/user", func(ctx iris.Context) {
    if versioning.Match(ctx, ">= 2.2.3") {
        // [处理器的版本逻辑出现在>= 2.2.3]
        return
    }
})
```
