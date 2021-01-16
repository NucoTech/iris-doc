# 路由

## 类型处理

顾名思义，类型处理用来处理程序不同类型的请求。

处理程序响应HTTP请求，将应答头和数据写入`Context.ResponseWriter()` 并将其返回。返回标志着这个请求的结束；在程序调用结束之后将无法继续使用`Context`的内容。

不管是HTTP客户端，HTTP协议版本，还有任何存在于客户端和iris服务之间的中间件，在结束`context.ResponseWriter()`之后将再也不被允许通过`Context.Request().Body`读取。所以警示处理器应该首先读取`Context.Request().Body`之后再进行请求回复。

除了读取消息正文，处理器并不被允许修改提供的`Context`。

如果有一个处理器崩溃，服务器 (处理器的调用者) 首先假定处理器崩溃的影响范围，并将它们隔离。它将重新恢复崩溃的处理器，并将堆栈跟踪记录到服务器错误日志并断开连接。

```go
type Handler func(iris.Context)
```

一旦一个处理器被注册，我们可以使用返回的Route实例为处理器提供一个名称，以便于调试或匹配视图中的相对路径。有关更多信息，请查看[反向查找](/ReverseLookups.md)部分。

## 行为

Iris的默认行为是使用`/api/user`这样的路径接受和注册路由,没有尾随斜杠。如果客户机试图访问`$your\u host/api/user/`则Iris路由器将自动永久重定向到`$your\u host/api/user`，以便由注册路由处理。这是设计api的现代方法。

但是，如果要对请求的资源禁用路径更正你可以通过配置iris配置中的 `iris.WithoutPathCorrection` 选项到你的`app.Run`，例如：

```go
// [app := iris.New...]
// [...]

app.Listen(":8080", iris.WithoutPathCorrection)
```

如果你想在`/adi/user`和`/api/user/`两个路径的路由中保持控制器的一致而不是通过重定向，你只需要通过`iris.WithoutPathCorrectionRedirection`选项来替代：

```go
app.Listen(":8080", iris.WithoutPathCorrectionRedirection)
```
