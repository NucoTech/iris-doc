# 响应记录器(Response recorder)

响应记录器是一个Iris中记录发送体, 状态码和消息头的特定的 `http.ResponseWriter` , 您可以在路由的请求处理程序链的任何处理程序中操作

1.在发送数据之前调用 `Context.Record()`
2.`Context.Record()`返回一个[响应记录器](https://godoc.org/github.com/kataras/iris/context#ResponseRecorder),  该方法可用于操作或者检索响应

`ResponseRecorder` 类型包含标准的Iris响应写入器(ResponseWriter)方法及以下方法

Body**返回**到目前为止追踪到的Body,不要使用它进行编辑

```go
Body() []byte
```

用这个清理body

```go
ResetBody()
```

使用 `Write/Writef/WriteString` 方法进行流写入并且使用 `SetBody/SetBodyString` 方法来**设置**body

```go
Write(contents []byte) (int, error)

Writef(format string, a ...interface{}) (n int, err error)

WriteString(s string) (n int, err error)

SetBody(b []byte)

SetBodyString(s string)
```

在调用 `Context.Record`方法之前, 将信息头重置到初始状态

```go
ResetHeaders()
```

清理所有的信息头

```go
ClearHeaders()
```

Reset方法重置响应体, 消息头和状态码

```go
Reset()
```

## 例子(Example)

在全局拦截器中记录操作日志

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    // 开始记录
    app.Use(func(ctx iris.Context) {
        ctx.Record()
        ctx.Next()
    })

    // 收集并且记录.
    app.Done(func(ctx iris.Context) {
        body := ctx.Recorder().Body()

        // 应该会打印"success".
        app.Logger().Infof("sent: %s", string(body))
    })
}
```

注册路由

```go
app.Get("/save", func(ctx iris.Context) {
    ctx.WriteString("success")
    ctx.Next() // 调用已完成的中间件
})
```

或者在主要信息头中消除依赖 `ctx.Next`, 可以如下修改Iris的处理程序的执行规则

```go
// 它适用于任何一个Party或者children
// 因此你可以创建一个路由 routes := app.Party("/path")
// 并且设置中间件, 规则和路由
app.SetExecutionRules(iris.ExecutionRules{ 
    Done: iris.ExecutionOptions{Force: true},
})

// [路由...]
app.Get("/save", func(ctx iris.Context) {
    ctx.WriteString("success")
})
```

除此之外, Iris为事务提供了一个全面的API, 运行[实例](https://github.com/kataras/iris/blob/master/_examples/response-writer/transactions/main.go)来了解更多
