# Flash Messages

有时, 您需要在某用户的请求间隙临时存储数据, 比如表单提交后的成功或错误信息, Iris的sessions包支持flash messages

正如我们在[Sessions](./Sessions/README.md)这一章中看到的, 您会这样初始化一个会话:

```go
import "github.com/kataras/iris/v12/sessions"

sess := sessions.New(sessions.Config{Cookie: "cookieName", ...})
```

```go
// [app := iris.New...]

app.Get("/path", func(ctx iris.Context) {
    session := sess.Start(ctx)
    // [...]
})
```

会话(Session)包含了下述储存、检索和移除flash messages的方法

```go
SetFlash(key string, value interface{})
HasFlash() bool
GetFlashes() map[string]interface{}

PeekFlash(key string) interface{}
GetFlash(key string) interface{}
GetFlashString(key string) string
GetFlashStringDefault(key string, defaultValue string) string

DeleteFlash(key string)
ClearFlashes()
```

方法的名称就解释了它的作用。例如, 如果您需要获得一条消息并在下一个请求之前删除它, 就使用 `GetFlash` 。当您只需要检索而不需删除时, 只需要使用 `PeekFlash`

> 小贴士: Flash messages并不会存储在数据库中
