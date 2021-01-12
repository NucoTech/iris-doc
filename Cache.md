# 缓存

有时, 提供静态内容的缓存路由很重要, 它可以使web应用表现得更快, 不必花时间为客户端重新构建响应

有两种方法可以实现HTTP缓存, 一种是在服务端存储每个处理程序的内容, 另一种是通过检查一些消息头信息并发送 `304 not modified` , 以便让浏览器或任何兼容的客户端自己处理缓存

Iris的 [Iris/cache](https://github.com/kataras/iris/tree/master/cache) 子包中的 `iris.Cache` 和 `iris.Cache304` 中间件提供了服务端和客户端的缓存

咱看看怎么写的

## Cache

`Cache` 是为下一个的处理程序提供服务端缓存的中间件, 可这样使用: `app.Get("/", iris.Cache, aboutHandler)`

`Cache` 接收一个参数: cache的过期时间, 如果过期值是一个无效值或<= 2s, 那么过期事件将会被 "cache-control's maxage" 消息头接管

所有类型的响应都可以被缓存, 无论是模板, json, 文本等

将其用于服务端的缓存, 请参阅 `Cache304` 了解更多更符合您需求的替代方法

```go
func Cache(expiration time.Duration) Handler
```

更多的选项和定制化使用[kataras/iris/cache.Cache](https://godoc.org/github.com/kataras/iris/cache#Cache), 它返回一个[可以添加或删除的 "Rules"结构](https://godoc.org/github.com/kataras/iris/cache/client#Handler)代替

## NoCache

`NoCache` 是一个覆盖了Cache-Control、Pragma和Expires消息头的中间件, 为了在浏览器的前进和后退中禁用缓存

这个中间件的一个妙用是在HTML路由上, 即使是浏览器页面上的后退和前进的按钮也会刷新页面

```go
func NoCache(Context)
```

相反的动作请看 `静态缓存(StaticCache)`

## StaticCache

`StaticCache` 给客户端发送 "Cache-Control" 和 "Expires" 消息头, 然后返回缓存静态文件的中间件, 它接受单个输入参数, "cacheDur" (一个时间), 用来计算到期时间的持续时间

如果 "cacheDur" <=0 , 那么在浏览器的前进和后退中, 它就返回 `NoCache` 中间件来禁用缓存

使用方法: `app.Use(iris.StaticCache(24 * time.Hour))` 或者 `app.Use(iris.StaticCache(-1))`

中间件是一个简单的处理程序, 也可以在另一个处理程序中调用, 例如:

```go
 cacheMiddleware := iris.StaticCache(...)

 func(ctx iris.Context){
  cacheMiddleware(ctx)
  [...]
}
```

## Cache304

当 "If-Modified-Since" 请求头(时间)在time.Now() + expiresEvery(总是与它们的UTC值比较)之前, `Cache304` 返回一个发送 `StatusNotModified` (304) 的中间件

与HTTP RCF兼容的客户端(所有的浏览器和像Postman这样的工具)将正确地处理缓存

该方法代替服务端缓存的唯一缺点是, 该方法发送304状态码, 而不是200, 因此, 如果你与其他微服务一起使用它, 你必须检查状态码以及有效的响应

开发人员可以通过观察系统目录的变化和使用基于文件修改日期的 `ctx.WriteWithExpiration` (带有 "modtime")方法来随意扩展这个方法的行为,  类似于 `HandleDir` (发送OK(200)状态且浏览器磁盘缓存代替304)

```go
func Cache304(expiresEvery time.Duration) Handler
```

可以在这里找更多的例子=>[链接](https://github.com/kataras/iris/tree/master/_examples/response-writer/cache)
