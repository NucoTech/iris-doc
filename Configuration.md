# 配置项(Configuration)

在[先前的章节](https://github.com/kataras/iris/wiki/Host)中我们已经学习了第一个输入参数`app.Run`, 现在我们看看下一个是什么。

让我们从基础开始。`iris.New`函数返回一个`iris.Application`。这个变量能通过`Configure(...iris.Configurator)`和`Run`的方法来进行配置。

第二部分配置, `app.Run/Listen`方法由一个或多个`iris.Configurator`来配置。每个`iris.Configurator`是一类: `func(app *iris.Application)`。用户自己配置的`iris.Configurator`也可以被传递进去来修改你的`*iris.Application`。

每一个核心的[Configuration](https://godoc.org/github.com/kataras/iris#Configuration)都内嵌了一个`iris.Configurator`, 例如`iris.WithoutStartupLog`, `iris.WithCharset("UTF-8")`, `iris.WithOptimizations`, 还有`iris.WithConfiguration(iris.Configuration{...})`等等函数。

每一个模块例如iris view engine, websockets, sessions, 以及其他的中间件都有它们独有的配置。

## 使用配置(UsingTheConfiguration)

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
// Tunnel 是一个隧道的配置结构体
type Tunnel struct {
    // Name是唯一一个必须的字段,
    // 它被用来建立和关闭隧道,
    // 例如: "MyApp"
    // 如果这个字段不为空的话, 当iris运行时就会建立一个ngrok的隧道。
    Name string `json:"name" yaml:"Name" toml:"Name"`
    // Addr是一个可选的字段, 它会在Iris的运行的过程中进行配置, 
    // 然而, 如果`iris.Raw`字段被配置过, 这个字段将会根据`hostname:port`
    // 表单进行配置, 因为框架不能检测出用户自己运行服务的地址。
    Addr string `json:"addr,omitempty" yaml:"Addr" toml:"Addr"`
}

// 隧道配置包含针对ngrok特性的设置。
// 需注意ngrok服务是否已经在主机上安装好了。
type TunnelingConfiguration struct {
    // AuthToken是一个可选字段, 用来给ngrok鉴权。
    // ngrok authtoken <YOUR_AUTHTOKEN>
    AuthToken string `json:"authToken,omitempty" yaml:"AuthToken" toml:"AuthToken"`

    // No...
    // Config是一个可选字段, 能被用来从文件系统加载ngrok的配置
    //
    // 如果你没有自定义目录存储配置文件, ngrok尝试从默认的路径
    // $HOME/.ngrok2/ngrok.yml读取文件。
    // 这个配置文件是可选的; 如果这个目录不存在，也不会报错。
    // Config string `json:"config,omitempty" yaml:"Config" toml:"Config"`

    // Bin是存放ngrok可执行文件的系统二进制文件目录。
    // 如果它是空的, 框架就会试图从系统环境变量去寻找它。
    Bin string `json:"bin,omitempty" yaml:"Bin" toml:"Bin"`

    // WebUIAddr是一个正在运行ngrok的web的接口地址。
    // 如果一个ngrok的示例在手动运行之前已经启动了, 
    // Iris会试图获取默认的web接口地址(http://127.0.0.1:4040), 
    // 然而如果使用了自定义的web接口, 这个字段就必须被设置成对应的地址
    // 例如: http://127.0.0.1:5050。
    WebInterface string `json:"webInterface,omitempty" yaml:"WebInterface" toml:"WebInterface"`

    // Region是一个可选项, 用来设置地区。默认为"us"
    // 以下变量有效
    // "us" for 美国(United States)
    // "eu" for 欧洲(Europe)
    // "ap" for 亚洲/太平洋(Asia/Pacific)
    // "au" for 澳大利亚(Australia)
    // "sa" for 南美(South America)
    // "jp" for 日本(Japan)
    // "in" for 印度(India)
    Region string `json:"region,omitempty" yaml:"Region" toml:"Region"`

    // Tunnels是网络隧道的集合。
    // 每一个Iris的程序对应一个网络隧道, 通常只需要一个。
    Tunnels []Tunnel `json:"tunnels" yaml:"Tunnels" toml:"Tunnels"`
}

// Configuration包含了一个Iris程序所必须的实例
// 所有的字段都是可选的, 默认的值会在一个普通的web程序上起作用
// 一个变量的配置通过`WithConfiguration`来设置。
// 
// Usage:
// conf := iris.Configuration{ ... }
// app := iris.New()
// app.Configure(iris.WithConfiguration(conf)) OR
// app.Run/Listen(..., iris.WithConfiguration(conf)).
type Configuration struct {
    // LogLevel用来输出程序的日志等级信息。
    // Logger, 默认大多数情况下用于记录构建状态, 但是它也可以用来调试在
    // 程序运行中报出的错误, 例如: 
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

    // Tunneling是一个可选项， 用来启用这个Iris项目实例的ngrok的http(s)隧道
    // 参见`WithTunneling`配置
    // Tunneling TunnelingConfiguration `json:"tunneling,omitempty" yaml:"Tunneling" toml:"Tunneling"`

    // IgnoreServerErrors will cause to ignore the matched "errors"
    // IgnoreServerErrors将从运行的主程序里忽略一个匹配到的"errors"
    // 这是一个字符串的切片, 而不是错误的切片。用户可以使用yaml或toml
    // 配置文件来注册这些错误。
    // 类比其余的那些配置字段
    // 参见`WithoutServerError(...)`。
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/http-server/listen-addr/omit-server-errors
    //
    // 默认是空切片。
    IgnoreServerErrors []string `json:"ignoreServerErrors,omitempty" yaml:"IgnoreServerErrors" toml:"IgnoreServerErrors"`

    // DisableStartupLog如果配置为true, 它将不会在服务启动时记录日志。
    // 默认为false。
    DisableStartupLog bool `json:"disableStartupLog,omitempty" yaml:"DisableStartupLog" toml:"DisableStartupLog"`
    // DisableInterruptHandler如果被设置成true, 当用户按下
    // control/cmd+C时程序不会优雅的退出。
    // 当你计划使用自己的方法去处理关闭服务请求时, 该字段可设置为true。
    // 
    // 默认为false。
    DisableInterruptHandler bool `json:"disableInterruptHandler,omitempty" yaml:"DisableInterruptHandler" toml:"DisableInterruptHandler"`

    // DisablePathCorrection字段用来配置是否允许自动纠正和重定向访问地址
    // 或者直接解析到注册的地址。
    // 例如: 如果请求/home/路径时, 该路径没有对应的路由, 
    // 路由就会自动检查/home是否存在, 如果存在会自动重定向到正确的/home
    // 路径。
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
    // 路径, 而不是返回一个404页面没有找到。

    // 默认为false。
    EnablePathIntelligence bool `json:"enablePathIntelligence,omitempty" yaml:"EnablePathIntelligence" toml:"EnablePathIntelligence"`
    // 当EnablePathEscape被设置为true的时候, 它会转义路径和命名参数(任意存在的参数)
    // 当你关掉它时: 
    // 接收'/'后面的参数
    // Request: http://localhost:8080/details/Project%2FDelta
    // ctx.Param("project")
    // 返回原始的命名变量: Project%2FDelta
    // 然后你可以通过net/url来手动解析它: 
    // projectName, _ := url.QueryUnescape(c.Param("project"))
    // 
    // 默认为false。
    EnablePathEscape bool `json:"enablePathEscape,omitempty" yaml:"EnablePathEscape" toml:"EnablePathEscape"`
    // ForceLowercaseRouting if enabled, converts all registered routes paths to lowercase
    // and it does lowercase the request path too for matching.
    // 
    // ForceLowercaseRouting如果配置为允许, 会将所有的注册路径转换成小写。
    // 同时也会将所有的请求路径转换成小写便于匹配。
    // 
    // 默认为false。
    ForceLowercaseRouting bool `json:"forceLowercaseRouting,omitempty" yaml:"ForceLowercaseRouting" toml:"ForceLowercaseRouting"`
    // FireMethodNotAllowed如果置为true, 路由会检查StatusMethodNotAllowed(405)
    // 请求会返回405错误而不是404
    // 默认为false。
    FireMethodNotAllowed bool `json:"fireMethodNotAllowed,omitempty" yaml:"FireMethodNotAllowed" toml:"FireMethodNotAllowed"`
    // DisableAutoFireStatusCode如果配置为true, 它会关闭http错误状态码处理程序
    // 自动调用`Context.StatusCode`获得的错误代码。
    // 默认情况下, 当调用"Context.StatusCode(errorCode)"时, 自定义的http错误
    // 处理程序将被触发。
    //
    // 默认为false。
    DisableAutoFireStatusCode bool `json:"disableAutoFireStatusCode,omitempty" yaml:"DisableAutoFireStatusCode" toml:"DisableAutoFireStatusCode"`
    // ResetOnFireErrorCode如果设置为true, 任何以前通过响应记录器或压缩写入的响应的
    // body或headers将被忽略, 路由器将触发注册的(或默认的)HTTP错误处理程序。
    // 参见`core/router/handler#FireErrorCode`和`Context.EndRequest`了解更多细节。
    // 了解更多: https://github.com/kataras/iris/issues/1531
    //
    // 默认为false。
    ResetOnFireErrorCode bool `json:"resetOnFireErrorCode,omitempty" yaml:"ResetOnFireErrorCode" toml:"ResetOnFireErrorCode"`

    // EnableOptimization当这个字段设置为true的时候, 
    // 程序会尽可能优化使得响应性能最优。
    // 
    // 默认为false。
    EnableOptimizations bool `json:"enableOptimizations,omitempty" yaml:"EnableOptimizations" toml:"EnableOptimizations"`
    // DisableBodyConsumptionOnUnmarshal管理上下文body读取/绑定的读取行为。
    // 如果设置为true, 那么它将禁止body的读取调用'context.UnmarshalBody/ReadJSON/ReadXML'
    // 默认情况下, 'io.ReadAll'用于从'context.Request.Body'中读取body, 返回一个
    // 'io.ReadCloser'。
    // 如果这个字段被设置成true, 会建立一个新的输入流来从请求的body中读取信息。
    // body和已经存在的数据不能被改变在'context.UnmarshalBody/ReadJSON/ReadXML'
    // 不能被调用之前。
    DisableBodyConsumptionOnUnmarshal bool `json:"disableBodyConsumptionOnUnmarshal,omitempty" yaml:"DisableBodyConsumptionOnUnmarshal" toml:"DisableBodyConsumptionOnUnmarshal"`
    // FireEmptyFormError如果被设置成true, 在请求数据是空的情况下`context.ReadBody/ReadForm`
    // 会返回一个`iris.ErrEmptyForm`。
    FireEmptyFormError bool `json:"fireEmptyFormError,omitempty" yaml:"FireEmptyFormError" yaml:"FireEmptyFormError"`

    // TimeFormat可以设置成任何你想解析的时间格式
    // 默认被设置成"Mon, 02 Jan 2006 15:04:05 GMT"。
    TimeFormat string `json:"timeFormat,omitempty" yaml:"TimeFormat" toml:"TimeFormat"`

    // Charset用于模板和其他响应的字符的编码。
    // 默认为"utf-8"
    Charset string `json:"charset,omitempty" yaml:"Charset" toml:"Charset"`

    // PostMaxMemory设置了最大化的一个客户端能向服务发送post的数据大小, 
    // 它不同于过度调用的可以被`context#SetMaxRequestBodySize`或
    // `iris#LimitRequestBodySize`改变的请求的body大小。
    // 默认为32MB或者32<<20
    PostMaxMemory int64 `json:"postMaxMemory" yaml:"PostMaxMemory" toml:"PostMaxMemory"`
    //  +----------------------------------------------------+
    //  | Context的变量用在变化特征上的键
    //  +----------------------------------------------------+
    // Context的变量用在变化特征上的键
    //
    // LocaleContextKey用于在本土得到当前的请求所在区域, 同时也包含一个翻译函数。
    // 默认为"iris.locale"。
    LocaleContextKey string `json:"localeContextKey,omitempty" yaml:"LocaleContextKey" toml:"LocaleContextKey"`
    // LanguageContextKey是一个语言能被中间件改变的上下文键。
    // 它比剩余的配置拥有最高的优先级, 如果它置空就会被忽略, 
    // 如果它被设置成了一个静态字段"default"或者默认的语言编码
    // 那么剩下的语言提取器将不会被调用, 默认的语言会被配置。
    // 和`Context.SetLanguage("el-GR")`一起用。
    // 参考`i18n.ExtractFunc`用更有组织的方式实现同一种特性。
    // 默认设置成"iris.locale.language"。
    LanguageContextKey string `json:"languageContextKey,omitempty" yaml:"LanguageContextKey" toml:"LanguageContextKey"`
    // VersionContextKey是一个API的版本能通过中间件`SetVersion`方法改变的上下文键。
    // 例如: `ctx.SetVersion("1.0, 1.1")`.
    // 默认是"iris.api.version"。
    VersionContextKey string `json:"versionContextKey" yaml:"VersionContextKey" toml:"VersionContextKey"`
    // GetViewLayoutContextKey是用来设置模板绑定数据从一个中间件或者处理器的布局的用户变量键。
    // 覆盖父进程或配置。
    // Defaults to "iris.ViewLayout"
    ViewLayoutContextKey string `json:"viewLayoutContextKey,omitempty" yaml:"ViewLayoutContextKey" toml:"ViewLayoutContextKey"`
    // GetViewDataContextKey是用来设置模板绑定数据从一个中间件或者处理器的用户变量键。
    // 默认设置成"iris.viewData"。
    ViewDataContextKey string `json:"viewDataContextKey,omitempty" yaml:"ViewDataContextKey" toml:"ViewDataContextKey"`
    
    // RemoteAddrHeaders是一个允许可以有效解析客户端IP的请求头名称字典。
    // 默认情况下, 没有"X-"标头被认为用于检索客户端IP地址是安全的,
    // 因为这些标头可以由客户端手动更改。
    // 但有时是有用的, 例如, 当在一个代理中,
    // 你想启用"X-Forward-For"或当你使用cloudflareCDN时你想启用
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
    // SSLProxyHeaders定义了一组表示有效的HTTPS请求的键值对
    // (参见`Context.IsSSL()`)。
    // 例如: `map[string]string{"X-Forwarded-Proto": "https"}`。
    // 
    // 默认为空字典。
    SSLProxyHeaders map[string]string `json:"sslProxyHeaders" yaml:"SSLProxyHeaders" toml:"SSLProxyHeaders"`
    // HostProxyHeaders定义了一个集合来保存客户端的代理主机名。
    // 参见`Context.Host()`了解更多。
    //
    // 默认为空字典。
    HostProxyHeaders map[string]bool `json:"hostProxyHeaders" yaml:"HostProxyHeaders" toml:"HostProxyHeaders"`
    // Other是用户自定义、动态的设置, 可以置空。
    // 这个字段只用于你来设置任何你想要的程序。
    //
    // 默认为空字典。
    Other map[string]interface{} `json:"other,omitempty" yaml:"Other" toml:"Other"`
}
```

## 从YAML导入(LoadFromYAML)

使用`iris.YAMl("path")`。

文件: iris.yml

```yml
FireMethodNotAllowed: true
DisableBodyConsumptionOnUnmarshal: true
TimeFormat: Mon, 01 Jan 2006 15:04:05 GMT
Charset: UTF-8
```

文件: main.go

```go
config := iris.WithConfiguration(iris.YAML("./iris.yml"))
app.Listen(":8080", config)
```

## 从TOML导入(LoadFromTOML)

使用`iris.TOML("path")`。

文件: iris.tml

```tml
FireMethodNotAllowed = true
DisableBodyConsumptionOnUnmarshal = false
TimeFormat = "Mon, 01 Jan 2006 15:04:05 GMT"
Charset = "UTF-8"

[Other]
    ServerName = "my fancy iris server"
    ServerOwner = "admin@example.com"
```

文件: main.go

```go
config := iris.WithConfiguration(iris.TOML("./iris.tml"))
app.Listen(":8080", config)
```

## 使用函数的方法(UsingTheFunctionalWay)

如同我们已经提到过的, 你可以使用任意数量的`iris.Configurator`作为`app.Run/Listen`的第二个参数。Iris提供了一个设置方法来设置每一个`iris.Configurator`
字段。

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

改变一部分配置字段并不困难。前缀: "With"或者"Without", 编辑器会帮助你浏览所有的配置项, 并能确保你的文档不会出现错误。

下列是一些用函数配置的例子

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

## 用户自定义变量(CustomValues)

`iris.Configuration`包含了一个字段名字叫做`Other map[string]interface{}`, 它可以接受任意自定义的`key:value`设置, 因此你可以用这个字段来按照你自己的需求来配置你自己的变量。

```go
app.Listen(":8080", 
    iris.WithOtherValue("ServerName", "my amazing iris server"),
    iris.WithOtherValue("ServerOwner", "admin@example.com"),
)
```

你可以通过`app.ConfigurationReadOnly`来访问配置字段

## 从Context中读取配置(AccessConfigurationFromContext)

```go
func (ctx iris.Context) {
    cfg := app.Application().ConfigurationReadOnly().GetOther()
    srvName := cfg["MyServerName"]
    srvOwner := cfg["ServerOwner"]
}
```
