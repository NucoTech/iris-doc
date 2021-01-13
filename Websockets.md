# Websockets

[Websocket协议](https://wikipedia.org/wiki/WebSocket)支持TCP连接中双向持久通信通道, 它可用于聊天、炒股、游戏等任何web应用中的实时功能

当您需要直接使用套接字连接的时候, 使用WebSockets, 比如, 您可能需要游戏实时具有最佳功能

首先, 阅读[kataras/neffos wiki](https://github.com/kataras/neffos/wiki)来掌握为net/http和Iris构建的新型websocket库

它可通过Iris预安装, 然而, 您可以通过以下shell命令来单独安装

```shell
go get github.com/kataras/neffos@latest
```

继续阅读如何将neffos websocket服务器注册到Iris应用中

使用websockets的完整例子可以在[这里](https://github.com/kataras/iris/tree/master/_examples/websocket)找到

[iris/websocket](https://github.com/kataras/iris/tree/master/websocket)子包仅包含特定于Iris的迁移和[neffos websocket framework](https://github.com/kataras/neffos)的帮助

例如, 要访问请求的 `Context` , 您可以调用内置于消息处理器/回调器的 `websocket.GetContext(Conn)` :

```go
// GetContext从websocket的连接中返回Iris Context.
func GetContext(c *neffos.Conn) Context
```

要注册一个websocket `neffos.Server` 到一个路由, 可使用 `websocket.Handler` 功能

```go
// IDGenerator是一个特定于Iris、服务新型连接的ID生成器
type IDGenerator func(Context) string

// Handler返回一个为Iris路由服务的Iris处理程序
// 接收neffos websocket服务器作为它的第一个输入参数, 
// 并且可选择一个特定于Iris的ID生成器作为第二个参数
func Handler(s *neffos.Server, IDGenerator ...IDGenerator) Handler
```

用法:

```go
import (
    "github.com/kataras/neffos"
    "github.com/kataras/iris/v12/websocket"
)

// [...]

onChat := func(ns *neffos.NSConn, msg neffos.Message) error {
    ctx := websocket.GetContext(ns.Conn)
    // [...]
    return nil
}

app := iris.New()
ws := neffos.New(websocket.DefaultGorillaUpgrader, neffos.Namespaces{
    "default": neffos.Events {
        "chat": onChat,
    },
})
app.Get("/websocket_endpoint", websocket.Handler(ws))
```

## Websocket控制器

Iris有通过Go结构体注册websocket事件的简单方法, websocket控制器是[MVC](/MVC.md)特性的一部分

Iris有专属的 `iris/mvc/Application.HandleWebsocket(v interface{}) *neffos.Struct` , 以便在现有的Iris MVC应用中注册控制器(为请求值和静态服务提供一个功能齐全的依赖注入容器)

```go
// HandleWebsocket处理特定的websocket控制器
// 它的导出方法是时间
// 如果有一个 "Namespace"字段 或者方法存在, 那么将设置命名空间
// 否则该控制器将使用空命名空间
//
// 注意:websocket控制器是在有连接连接到命名空间后才能注册并运行的
// 并且它不能在此状态下发送HTTP响应
// 然而, 所有的静态和动态依赖的行为都是预期内的
func (*mvc.Application) HandleWebsocket(controller interface{}) *neffos.Struct
```

咱们看一个使用实例, 我们想把 `OnNamespaceConnected` 、 `OnNamespaceDisconnect` 内置事件与自定义的 `"OnChat"` 事件与控制器方法绑定

1.我们通过声明NSConn类型字段为 `stateless` 来创建控制器, 并编写我们所需要的方法

```go
type websocketController struct {
    *neffos.NSConn `stateless:"true"`
    Namespace string

    Logger MyLoggerInterface
}

func (c *websocketController) OnNamespaceConnected(msg neffos.Message) error {
    return nil
}

func (c *websocketController) OnNamespaceDisconnect(msg neffos.Message) error {
    return nil
}

func (c *websocketController) OnChat(msg neffos.Message) error {
    return nil
}
```

Iris足够聪明去捕获 `Namespace string` 结构体字段, 然后用它为命名空间注册视为事件的控制器方法, 或者您可以创建一个 `Namespace() string { return "default" }` 的控制器方法或将 `HandleWebsocket` 的返回值赋予 `.SetNamespace("default")` , 取决于您

2.我们将MVC应用的目标初始化为websocket结点, 就像过去对HTTP路由的常规HTTP控制器那样

```go
import (
    // [...]
    "github.com/kataras/iris/v12/mvc"
)
// [app := iris.New...]

mvcApp := mvc.New(app.Party("/websocket_endpoint"))
```

3.我们依赖注册项(如果有的话)

```go
mvcApp.Register(
    &prefixedLogger{prefix: "DEV"},
)
```

4.我们注册一个或多个websocket控制器, 每个websocket控制器映射到一个命名空间(一个就够了, 大多数情况下你不需要更多, 但这取决于你的应用需要和要求)

```go
mvcApp.HandleWebsocket(&websocketController{Namespace: "default"})
```

接下来, 我们继续将mvc应用映射为websocket服务器的连接处理程序(你可以调用 `neffos.JoinConnHandlers(mvcApp1, mvcApp2)` , 使得每一个websocket服务器不止一个MVC应用)

```go
websocketServer := neffos.New(websocket.DefaultGorillaUpgrader, mvcApp)
```

6.最后一步是通过一个普通的 `.Get` 方法将该服务器注册到我们的结点

```go
mvcApp.Router.Get("/", websocket.Handler(websocketServer))
```
