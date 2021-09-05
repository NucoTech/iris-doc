# 子路由 (Routing subdomains)

Iris为单个应用程序提供了已知的最简单的子域名注册形式。当然, 您可以使用nginx或caddy进行生产管理

子域分为两类:**静态**和**动态/通配符**

- 静态: 当你知道子域名时, 例如: `analytics.mydomain.com`
- 通配符: 当你不知道子域, 但你知道它在一个特定的子域或根域之前, 例如: `user_created.mydomain.com` , `otheruser.mydomain.com` 像 `username.github.io`

子域方法返回一个新 `Party` , 负责注册到这个特定 "子域" 的路由

与常规方的唯一区别是, 如果从子方调用, 则子域将被添加到路径之前, 而不是追加。因此, 如果 `app.Subdomain("admin"). subdomain("panel")` , 那么结果是: `"panel.admin"`

```go
Subdomain(subdomain string, middleware ...Handler) Party
```

`WildcardSubdomain` 方法返回一个新的 `Party` , 负责注册到一个动态的通配符子域的路由。动态子域是一种可以处理任何子域请求的子域。服务器将接受任何子域(如果没有找到静态子域), 它将搜索并执行该 `Party` 的处理程序

```go
WildcardSubdomain(middleware ...Handler) Party
```

实例代码:

```go
// [app := iris.New...]
admin := app.Subdomain("admin")

// admin.mydomain.com
admin.Get("/", func(ctx iris.Context) {
    ctx.Writef("INDEX FROM admin.mydomain.com")
})

// admin.mydomain.com/hey
admin.Get("/hey", func(ctx iris.Context) {
    ctx.Writef("HEY FROM admin.mydomain.com/hey")
})

// [这里是其他路由...]

app.Listen("mydomain.com:80")
```

对于本地开发, 你必须编辑你的主机, 例如在windows操作系统中打开 C:\Windows\System32\Drivers\etc\hosts 文件并且添加: 

```json
127.0.0.1 mydomain.com
127.0.0.1 admin.mydomain.com
```

为了证明子域名像任何其他常规方一样工作, 你也可以使用下面的替代方法注册子域名:

```go
adminSubdomain:= app.Party("admin.")
// 或者
adminAnalayticsSubdomain := app.Party("admin.analytics.")
// 或者是动态的:
anySubdomain := app.Party("*.")
```

还有一个 `iris.Application` 方法, 它允许为子域注册一个全局重定向规则

`SubdomainRedirect` 设置(如果使用多次, 则添加)一个路由器包装器, 在路由的处理程序执行之前尽快将一个(子)域重定向(statusmoved永久)到另一个子域或根域

它接收两个参数, 它们是from和to/target, 'from'也可以是一个通配符子域(app.WildcardSubdomain())由于明显的原因, 'to'不能是通配符, 当'to'不是根域时, 'from'可以是根域(应用程序), 反之亦然

```go
SubdomainRedirect(from, to Party) Party
```

使用方法:

```go
www := app.Subdomain("www")
app.SubdomainRedirect(app, www)
```

上面将重定向所有 `http(s)://mydomain.com/%anypath%` 到 `http(s)://www.mydomain.com/%anypath%`

在处理子域时, `Context` 提供了四种主要方法, 它们可能对您有帮助

```go
// Host返回当前url的主机部分
Host() string
// Subdomain返回此请求的子域(如果有的话)
// 注意, 这是一个快速的方法, 它并不能覆盖所有的情况
Subdomain() (subdomain string)
// IsWWW, 如果当前子域(如果有)是www, 则返回true
IsWWW() bool
// FullRqeuestURI返回完整的URI, 包括方案、主机和相对请求的路径/资源
FullRequestURI() string
```

使用方法:

```go
func info(ctx iris.Context) {
    method := ctx.Method()
    subdomain := ctx.Subdomain()
    path := ctx.Path()

    ctx.Writef("\nInfo\n\n")
    ctx.Writef("Method: %s\nSubdomain: %s\nPath: %s", method, subdomain, path)
}
```
