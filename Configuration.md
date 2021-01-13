# 配置项(Configuration)

在[先前的章节](https://github.com/kataras/iris/wiki/Host)中我们已经学习了`app.Run`的第一个输入参数, 现在我们看看第二个是什么。

让我们从基础开始。`iris.New`函数返回一个`iris.Application`。这个变量能通过`Configure(...iris.Configurator)`和`Run`的方法来进行配置。

第二个可选, `app.Run/Listen`方法的可变参数接受一个或多个`iris.Configurator`。`iris.Configurator`是`func(app *iris.Application)`类型。自定义的`iris.Configurator`也可以被传递以修改你的`*iris.Application`。

每一个核心的[Configuration](https://godoc.org/github.com/kataras/iris#Configuration)字段都内置了`iris.Configurator`们, 例如`iris.WithoutStartupLog`, `iris.WithCharset("UTF-8")`, `iris.WithOptimizations`, 还有`iris.WithConfiguration(iris.Configuration{...})`等函数。

每一个模块例如iris view engine, websockets, sessions, 以及其他的中间件都有它们独有的配置和选项。

## 使用[配置](https://godoc.org/github.com/kataras/iris#Configuration)

所有的`iris.Configuration`字段都被预设成了最常见的情况。如果你在任何时候想使用自定义的`iris.Configuration`, 可以调用app.Configure(accepts iris.Configurator)来进行你的配置。

```go
config := iris.WithConfiguration(iris.Configuration {
    DisableStartupLog: true,
    Optimizations: true,
    Charset: "UTF-8",
})

app.Listen(":8080", config)
```

下列是所有有效的配置

```go
// Tunnel是TunnelingConfiguration结构体的Tunnels字段
type Tunnel struct {
    // Name是唯一一个需要的字段,
    // 它被用来建立和关闭隧道,
    // 例如: "MyApp"
    // 如果这个字段不为空的话, 当iris运行时就会生成一个ngrok的隧道。
    Name string `json:"name" yaml:"Name" toml:"Name"`
    // Addr是一个基础的可选字段, 它会内置在Iris运行的时进行设置, 
    // 然而, 如果用到了`iris.Raw`字段, 这个字段将会根据`hostname:port`
    // 表单进行配置, 因为框架不能感知在自定义运行时服务的地址
    Addr string `json:"addr,omitempty" yaml:"Addr" toml:"Addr"`
}

// TunnelingConfiguration包含通过ngrok特性的可选的tunneling的设置。
// 需注意host上是否已经安装了ngrok服务。
type TunnelingConfiguration struct {
    // AuthToken是一个可选字段, 用来给ngrok鉴权。
    // ngrok authtoken <YOUR_AUTHTOKEN>
    AuthToken string `json:"authToken,omitempty" yaml:"AuthToken" toml:"AuthToken"`

    // No...
    // Config是可选的, 能被用来从文件系统加载ngrok的配置
    //
    // 如果你没有指定存放配置文件的位置, ngrok尝试从默认的路径
    // $HOME/.ngrok2/ngrok.yml读取文件。
    // 这个配置文件是可选的; 如果这个目录不存在，也不会报错。
    // Config string `json:"config,omitempty" yaml:"Config" toml:"Config"`

    // Bin是存放ngrok可执行文件的系统二进制文件目录。
    // 如果它是空的, 框架就会试图从系统环境变量去寻找它。
    Bin string `json:"bin,omitempty" yaml:"Bin" toml:"Bin"`

    // WebUIAddr是一个正在运行ngrok实例的web的接口地址。
    // 在试图手动启动ngrok实例之前, 
    // Iris会试图获取默认的web接口地址(http://127.0.0.1:4040)来确定它是否在运行。 
    // 然而如果使用了自定义的web接口, 这个字段就必须被设置
    // 例如: http://127.0.0.1:5050。
    WebInterface string `json:"webInterface,omitempty" yaml:"WebInterface" toml:"WebInterface"`

    // Region是一个可选项, 用来设置地区。默认为"us"
    // 以下变量有效
    // "us"代表美国(United States)
    // "eu"代表欧洲(Europe)
    // "ap"代表亚洲/太平洋(Asia/Pacific)
    // "au"代表澳大利亚(Australia)
    // "sa"代表南美(South America)
    // "jp"代表日本(Japan)
    // "in"代表印度(India)
    Region string `json:"region,omitempty" yaml:"Region" toml:"Region"`

    // Tunnels是网络隧道的集合。
    // 每个应用程序的每个Iris Host有一个隧道, 通常只需要一个。
    Tunnels []Tunnel `json:"tunnels" yaml:"Tunnels" toml:"Tunnels"`
}

// Configuration包含了Iris应用程序实例所必要的设置
// 所有的字段都是可选的, 默认的值会在常规的web程序上起作用
// 配置项的值可以通过`WithConfiguration`来传递。
// 
// Usage:
// conf := iris.Configuration{ ... }
// app := iris.New()
// app.Configure(iris.WithConfiguration(conf)) OR
// app.Run/Listen(..., iris.WithConfiguration(conf))。
type Configuration struct {
    // LogLevel是应用程序输出消息时应该使用的日志等级。
    // Logger, 默认大多数情况用于构建态, 但是它也可用在调试
    // 程序运行时的报错信息, 例如: 
    // 当客户端发送一些异常的数据结构体时(例如: Context.JSON/JSONP/XML...)
    //
    // 默认是"info"。以下值有效:
    // * "disable"
    // * "fatal"
    // * "error"
    // * "warn"
    // * "info"
    // * "debug"
    LogLevel string `json:"logLevel" yaml:"LogLevel" toml:"LogLevel" env:"LOG_LEVEL"`

    // Tunneling是一个可选项， 用来启用这个Iris应用程序实例的ngrok http(s)隧道
    // 参见`WithTunneling`配置器(Configurator)
    Tunneling TunnelingConfiguration `json:"tunneling,omitempty" yaml:"Tunneling" toml:"Tunneling"`

    // IgnoreServerErrors将从主程序的`Run`函数里忽略匹配到的"errors"
    // 这是一个字符串的切片, 而不是错误的切片。
    // 用户可以跟其他的配置字段一样使用yaml或者toml配置文件注册这些错误。
    // 参见`WithoutServerError(...)`函数。
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/http-server/listen-addr/omit-server-errors
    //
    // 默认是空切片。
    IgnoreServerErrors []string `json:"ignoreServerErrors,omitempty" yaml:"IgnoreServerErrors" toml:"IgnoreServerErrors"`

    // DisableStartupLog如果配置为true, 在服务器启动时会关闭写入。
    // 默认为false。
    DisableStartupLog bool `json:"disableStartupLog,omitempty" yaml:"DisableStartupLog" toml:"DisableStartupLog"`
    // DisableInterruptHandler如果被设置成true, 当用户按下
    // control/cmd+C时, 程序不会自动优雅的退出。
    // 当你准备自行通过自定义的host.Task处理请求时, 该字段应设置为true。
    // 
    // 默认为false。
    DisableInterruptHandler bool `json:"disableInterruptHandler,omitempty" yaml:"DisableInterruptHandler" toml:"DisableInterruptHandler"`

    // DisablePathCorrection用来禁止纠正和重定向, 
    // 或者直接将请求地址处理输出到已注册的地址
    // 例如: 如果请求/home/路径时, 该路由没有对应的处理, 
    // 路由就会自动检查 /home 处理是否存在, 如果存在(永久的)重定向到正确的路径 /home
    // 参见DisablePathCorrectionRedirection来允许直接处理访问请求而
    // 不是重定向访问。
    // 
    // 默认为false。
    DisablePathCorrection bool `json:"disablePathCorrection,omitempty" yaml:"DisablePathCorrection" toml:"DisablePathCorrection"` 
    // DisablePathCorrectionRedirection在任何时候配置都会起作用。
    // DisablePathCorrection设置为false时, 
    // 如果DisablePathCorrectionRedirection被设置为true, 
    // 它会处理匹配到的这个请求, 将请求末尾的("/")删去而不是发送一个重定向状态
    // 
    // 默认为false。
    DisablePathCorrectionRedirection bool `json:"disablePathCorrectionRedirection,omitempty" yaml:"DisablePathCorrectionRedirection" toml:"DisablePathCorrectionRedirection"`
    // EnablePathIntelligence如果被设置为true,
    // 路由会把没有找到页面的HTTP的GET方式请求, 重定向为任何与它最相似的路径。
    // 例如: 
    // 你在"/contact"路径注册了一个路由
    // 客户端试图访问"/cont"路径, 这个访问将会被自动修复并重定向至"/contact"
    // 路径, 而不是返回404 not found。

    // 默认为false。
    EnablePathIntelligence bool `json:"enablePathIntelligence,omitempty" yaml:"EnablePathIntelligence" toml:"EnablePathIntelligence"`
    // 当EnablePathEscape被设置为true的时候, 它会转义路径和命名参数(任意存在的参数)
    // 当你关掉它时: 
    // 接收带有'/'的参数
    // Request: http://localhost:8080/details/Project%2FDelta
    // ctx.Param("project")
    // 返回原始的命名变量: Project%2FDelta
    // 你可以通过net/url来手动解析它: 
    // projectName, _ := url.QueryUnescape(c.Param("project"))
    // 
    // 默认为false。
    EnablePathEscape bool `json:"enablePathEscape,omitempty" yaml:"EnablePathEscape" toml:"EnablePathEscape"`
    // ForceLowercaseRouting如果配置为允许, 会将所有的注册路径转换成小写。
    // 同时也会将所有的请求路径转换成小写便于匹配。
    // 
    // 默认为false。
    ForceLowercaseRouting bool `json:"forceLowercaseRouting,omitempty" yaml:"ForceLowercaseRouting" toml:"ForceLowercaseRouting"`
    // FireMethodNotAllowed如果置为true, 路由会检查StatusMethodNotAllowed(405)
    // 请求会返回405错误而不是404
    // 默认为false。
    FireMethodNotAllowed bool `json:"fireMethodNotAllowed,omitempty" yaml:"FireMethodNotAllowed" toml:"FireMethodNotAllowed"`
    // DisableAutoFireStatusCode如果配置为true, 它会关闭http错误状态码处理
    // 自动从`Context.StatusCode`调用输出错误代码。
    // 默认情况下, 当调用"Context.StatusCode(errorCode)"时, 自定义的http错误
    // 处理程序将被触发。
    //
    // 默认为false。
    DisableAutoFireStatusCode bool `json:"disableAutoFireStatusCode,omitempty" yaml:"DisableAutoFireStatusCode" toml:"DisableAutoFireStatusCode"`
    // ResetOnFireErrorCode如果设置为true, 任何以前通过响应记录器(response recorder)或gzip写入的
    // 响应体或头(header)将会被忽略, 
    // 路由器触发已注册的(或默认的)HTTP错误处理作为替代。
    // 参见`core/router/handler#FireErrorCode`和`Context.EndRequest`了解更多细节。
    // 阅读更多: https://github.com/kataras/iris/issues/1531
    //
    // 默认为false。
    ResetOnFireErrorCode bool `json:"resetOnFireErrorCode,omitempty" yaml:"ResetOnFireErrorCode" toml:"ResetOnFireErrorCode"`

    // EnableOptimization当这个字段设置为true的时候, 
    // 程序会尽可能优化使得响应性能最优。
    // 
    // 默认为false。
    EnableOptimizations bool `json:"enableOptimizations,omitempty" yaml:"EnableOptimizations" toml:"EnableOptimizations"`
    // DisableBodyConsumptionOnUnmarshal 管理上下文body读取器/绑定器的读取行为。
    // 如果设置为true, 那么它将禁止body的读取调用'context.UnmarshalBody/ReadJSON/ReadXML'
    // 默认情况下, 'io.ReadAll'用于从'context.Request.Body'中读取body, 一个
    // 'io.ReadCloser'。
    // 如果这个字段被设置成true, 会建立一个新的缓冲区(buffer)来从请求的body中读取信息。
    // 在'context.UnmarshalBody/ReadJSON/ReadXML'被消耗之前
    // 已存在的数据和body不能被改变。
    DisableBodyConsumptionOnUnmarshal bool `json:"disableBodyConsumptionOnUnmarshal,omitempty" yaml:"DisableBodyConsumptionOnUnmarshal" toml:"DisableBodyConsumptionOnUnmarshal"`
    // FireEmptyFormError 如果被设置成true, 在请求的form data是空的情况下
    // `context.ReadBody/ReadForm` 会返回一个`iris.ErrEmptyForm`。
    FireEmptyFormError bool `json:"fireEmptyFormError,omitempty" yaml:"FireEmptyFormError" yaml:"FireEmptyFormError"`

    // TimeFormat 为各种时间解析提供时间格式化
    // 默认被设置成"Mon, 02 Jan 2006 15:04:05 GMT"。
    TimeFormat string `json:"timeFormat,omitempty" yaml:"TimeFormat" toml:"TimeFormat"`

    // Charset 为用于模板和其他响应中使用的字符进行编码。
    // 默认为"utf-8"
    Charset string `json:"charset,omitempty" yaml:"Charset" toml:"Charset"`

    // PostMaxMemory 设置了客户端能向服务发送post的最大数据大小, 
    // 与超额请求体大小不同的是, 这个可以被、
    // `context#SetMaxRequestBodySize`或 iris#LimitRequestBodySize`修改。
    // 默认为 32MB 或者 32 << 20 如果你喜欢的话
    PostMaxMemory int64 `json:"postMaxMemory" yaml:"PostMaxMemory" toml:"PostMaxMemory"`
    //  +----------------------------------------------------+
    //  | 被用在各种特性上的上下文键值
    //  +----------------------------------------------------+
    // 被用在各种特性上的上下文键
    //
    // LocaleContextKey 用于国际化获取当前请求的本土化, 其中包含了一个翻译函数。
    // 默认为"iris.locale"。
    LocaleContextKey string `json:"localeContextKey,omitempty" yaml:"LocaleContextKey" toml:"LocaleContextKey"`
    // LanguageContextKey 是一个中间件可以修改语言的上下文键。
    // 它比其它的配置拥有最高的优先级, 如果它置空就会被忽略, 
    // 如果它被设置成了一个静态字符串"default"或者默认的语言的代码
    // 那么其余的语言提取器将不会被调用, 默认的语言会被配置。
    // 使用`Context.SetLanguage("el-GR")`。
    // 参考`i18n.ExtractFunc`用更有组织的方式实现同一种特性。
    // 默认设置成"iris.locale.language"。
    LanguageContextKey string `json:"languageContextKey,omitempty" yaml:"LanguageContextKey" toml:"LanguageContextKey"`
    // VersionContextKey 是能通过中间件'SetVersion'方法修改的API版本上下文键。
    // 例如: `ctx.SetVersion("1.0, 1.1")`.
    // 默认是"iris.api.version"。
    VersionContextKey string `json:"versionContextKey" yaml:"VersionContextKey" toml:"VersionContextKey"`
    // GetViewLayoutContextKey 是被用来从中间件或者主处理器中设置模板数据布局的上下文中用户(设置)
    // 值的键覆盖父级或者配置中的
    // Defaults to "iris.ViewLayout"
    ViewLayoutContextKey string `json:"viewLayoutContextKey,omitempty" yaml:"ViewLayoutContextKey" toml:"ViewLayoutContextKey"`
    // GetViewDataContextKey 是被用来从中间件或者主处理器中绑定模板数据的上下文中用户(设置)值的键。
    // 默认设置成"iris.viewData"。
    ViewDataContextKey string `json:"viewDataContextKey,omitempty" yaml:"ViewDataContextKey" toml:"ViewDataContextKey"`
    // RemoteAddrHeaders是允许的请求头名称, 可以有效解析客户端的IP。
    // 默认情况下, 没有"X-"标头被认为用于检索客户端IP地址是安全的,
    // 因为这些标头可以由客户端手动更改。
    // 但有时是有用的, 例如, 当经过一层代理,
    // 你想启用"X-Forward-For", 或当使用cloudflare你想启用
    // "CF-Connected-IP", 在需要时, 
    // 你可以允许'ctx.RemoteAddr()'使用客户端可能发送的任何header。
    //
    // 默认是一个空的字典, 一种示例用法如下:
    // RemoteAddrHeaders {
    //  "X-Real-Ip":             true,
    //  "X-Forwarded-For":       true,
    //  "CF-Connecting-IP":      true,
    // }
    //
    // 参见`context.RemoteAddr()`了解更多。
    RemoteAddrHeaders map[string]bool `json:"remoteAddrHeaders,omitempty" yaml:"RemoteAddrHeaders" toml:"RemoteAddrHeaders"`
    // RemoteAddrPrivateSubnets定义了私有子网地址。
    // 它们通常要和从`RemoteAddrHeaders`或
    // `Context.Request.RemoteAddr`获取到的IP地址进行对比。
    // 更多细节详见: https://github.com/kataras/iris/issues/1453
    // 默认配置:
    // {
    // Start: net.ParseIP("10.0.0.0"),
    // End:   net.ParseIP("10.255.255.255"),
    // },
    // {
    // Start: net.ParseIP("100.64.0.0"),
    // End:   net.ParseIP("100.127.255.255"),
    // },
    // {
    // Start: net.ParseIP("172.16.0.0"),
    // End:   net.ParseIP("172.31.255.255"),
    // },
    // {
    // Start: net.ParseIP("192.0.0.0"),
    // End:   net.ParseIP("192.0.0.255"),
    // },
    // {
    // Start: net.ParseIP("192.168.0.0"),
    // End:   net.ParseIP("192.168.255.255"),
    // },
    // {
    // Start: net.ParseIP("198.18.0.0"),
    // End:   net.ParseIP("198.19.255.255"),
    // }
    //
    // 查看`Context.RemoteAddr()`了解更多。
    RemoteAddrPrivateSubnets []netutil.IPRange `json:"remoteAddrPrivateSubnets" yaml:"RemoteAddrPrivateSubnets" toml:"RemoteAddrPrivateSubnets"`
    // SSLProxyHeaders定义了header键值对集合表示合法的HTTPS请求
    // (参见`Context.IsSSL()`)。
    // 例如: `map[string]string{"X-Forwarded-Proto": "https"}`。
    // 
    // 默认为空字典。
    SSLProxyHeaders map[string]string `json:"sslProxyHeaders" yaml:"SSLProxyHeaders" toml:"SSLProxyHeaders"`
    // HostProxyHeaders为客户端定义了headers的集合保存代理过(proxied)的主机名。
    // 参见`Context.Host()`了解更多。
    //
    // 默认为空字典。
    HostProxyHeaders map[string]bool `json:"hostProxyHeaders" yaml:"HostProxyHeaders" toml:"HostProxyHeaders"`
    // Other是自定义、动态的设置, 可以为空。
    // 这个字段只用于设置你在应用程序中的一些选项。
    //
    // 默认为空字典。
    Other map[string]interface{} `json:"other,omitempty" yaml:"Other" toml:"Other"`
}
```

## 从[YAML](https://yaml.org/)导入

使用`iris.YAMl("path")`。

文件: **iris.yml**

```yml
FireMethodNotAllowed: true
DisableBodyConsumptionOnUnmarshal: true
TimeFormat: Mon, 01 Jan 2006 15:04:05 GMT
Charset: UTF-8
```

文件: **main.go**

```go
config := iris.WithConfiguration(iris.YAML("./iris.yml"))
app.Listen(":8080", config)
```

## 从[TOML](https://github.com/toml-lang/toml)导入

使用`iris.TOML("path")`。

文件: **iris.tml**

```tml
FireMethodNotAllowed = true
DisableBodyConsumptionOnUnmarshal = false
TimeFormat = "Mon, 01 Jan 2006 15:04:05 GMT"
Charset = "UTF-8"

[Other]
    ServerName = "my fancy iris server"
    ServerOwner = "admin@example.com"
```

文件: **main.go**

```go
config := iris.WithConfiguration(iris.TOML("./iris.tml"))
app.Listen(":8080", config)
```

## 使用函数方式

如同我们已经提到过的, 你可以在`app.Run/Listen`的第二个参数传递任何数量的`iris.Configrator`。Iris为每一个`iris.Configurator`
字段提供了一个选项。

```go
app.Listen(":8080", iris.WithoutInterruptHandler,
    iris.WithoutBodyConsumptionOnUnmarshal,
    iris.WithoutAutoFireStatusCode,
    iris.WithLowercaseRouting,
    iris.WithPathIntelligence,
    iris.WithOptimizations,
    iris.WithTimeFormat("Mon, 01 Jan 2006 15:04:05 GMT"),
)
```

当你想去改配置文件中的一些字段是极好的。前缀: "With"或者"Without", 代码编辑器会帮助你浏览所有的配置项, 来确保文档没有问题。

函数选项列表

```go
var (
    WithGlobalConfiguration Configurator
    WithoutStartupLog, WithoutBanner Configurator
    WithoutInterruptHandler Configurator
    WithoutPathCorrection Configurator
    WithPathIntelligence Configurator
    WithoutPathCorrectionRedirection Configurator
    WithoutBodyConsumptionOnUnmarshal Configurator
    WithEmptyFormError Configurator
    WithPathEscape Configurator
    WithLowercaseRouting Configurator
    WithOptimizations Configurator
    WithFireMethodNotAllowed Configurator
    WithoutAutoFireStatusCode Configurator
    WithResetOnFireErrorCode Configurator
    WithTunneling Configurator
)

func WithLogLevel(level string) Configurator
func WithoutServerError(errors ...error) Configurator
func WithTimeFormat(timeformat string) Configurator
func WithCharset(charset string) Configurator
func WithPostMaxMemory(limit int64) Configurator
func WithRemoteAddrHeader(header ...string) Configurator
func WithoutRemoteAddrHeader(headerName string) Configurator
func WithRemoteAddrPrivateSubnet(startIP, endIP string) Configurator
func WithSSLProxyHeader(headerKey, headerValue string) Configurator
func WithHostProxyHeader(headers ...string) Configurator
func WithOtherValue(key string, val interface{}) Configurator
func WithSitemap(startURL string) Configurator
```

## 自定义值

`iris.Configuration`包含了一个叫`Other map[string]interface{}`的字段, 它可以接受任意自定义的键值对(`key:value`)选项, 因此你可以根据自己的应用程序需求使用这个字段配置指定值。

```go
app.Listen(":8080", 
    iris.WithOtherValue("ServerName", "my amazing iris server"),
    iris.WithOtherValue("ServerOwner", "admin@example.com"),
)
```

你可以通过`app.ConfigurationReadOnly`来访问那些字段

## 从上下文中获取配置

```go
func (ctx iris.Context) {
    cfg := app.Application().ConfigurationReadOnly().GetOther()
    srvName := cfg["MyServerName"]
    srvOwner := cfg["ServerOwner"]
}
```
