# 上下文方法(All context Methods)

这里完整列出了 `iris.Context` 提供的方法

```go
type Context interface {
    // 在请求结束之前, 会在这个上下文延迟执行一个处理程序
    // `StopExecution` 不会影响延迟处理程序的执行
    // "h" 会先于 `FireErrorCode` 执行(当请求状态码为不成功时).
    Defer(Handler)

    // ResponseWriter会如预期返回一个http.ResponseWriter兼容的响应写入器(response writer)
    ResponseWriter() ResponseWriter
    // ResetResponseWriter应更改或更新上下文的响应写入器
    ResetResponseWriter(ResponseWriter)

    // Request会如预期返回一个原始的 *http.Request
    Request() *http.Request
    // ResetRequest设置上下文请求,
    // 将 std *http.Request#WithContext()
    // 创建的新请求存储到Iris的上下文中是很有用的
    // 当您出于某种原因想要完全覆盖 *http.Request时, 请使用`ResetRequest`
    // 注意: 当你只想更改其中的某个字段时, 您可以使用Request(),
    // 它会返回一个指向Request的指针
    // 这样不需要完全覆盖就可以产生作用
    // 使用本地http处理程序, 它使用标准的 "Context" 包来获取值
    // 代替Iris的Context#Values()
    // r := ctx.Request()
    // stdCtx := context.WithValue(r.Context(), key, val)
    // ctx.ResetRequest(r.WithContext(stdCtx)).
    ResetRequest(r *http.Request)

    // SetCurrentRoutes在内部设置路由
    // 参见 `GetCurrentRoute()` 方法
    // 它被路由器初始化
    // 参见 `Exec` 或  `SetHandlers/AddHandler` 方法来模拟一个请求
    SetCurrentRoute(route RouteReadOnly)
    // GetCurrentRoute返回注册到这个请求路径的 "只读" 路由
    GetCurrentRoute() RouteReadOnly

    // Do调用SetGandlers(处理程序)并且执行第一个处理程序, 
    // 处理程序不能为空
    // 它会被路由使用, 开发者可以用它代替并立即执行处理程序
    Do(Handlers)
    // AddHandler可以在服务时间向当前请求添加处理程序, 
    // 这些处理程序不会在路由器上持久化
    //
    // 路由器正在调用这个函数来添加路由的处理程序
    // 如果调用AddHandler, 那么处理程序将被插入到已经定义的
    // 路由程序的末尾
    AddHandler(...Handler)
    // SetHandlers更新所有的处理程序
    SetHandlers(Handlers)
    // Handlers保持跟踪当前的处理程序
    Handlers() Handlers
    // HandlerIndex设置当前上下文处理程序链的当前索引
    // 如果n < 0或当前处理器长度为0, 那么它只返回当前处理器索引
    // 而不改变当前索引 
    HandlerIndex(n int) (currentIndex int)
    // Proceed是检查特定处理程序是否已经被执行的另一种方法
    // 在里面被称为 `ctx.Next`
    // 只有当你在另一个处理程序中运行一个处理程序时, 
    // 这个函数才起作用, 它只检查before索引和after索引
    //一个用例是当你想要想在控制器的 `BeginRequest` 中执行一个中间件时
    // 调用其中的 `ctx.Next` 方法, 控制器将整个流程
    // (开始请求, 方法处理程序, 结束请求)视为一个处理程序,
    // 如果从 `BeginRequest` 中调用 `ctx.Next` 将不会反射到方法处理程序
    //
    // 虽然 `BeginRequest` 不应该被用来调用其他处理程序, 但引入 `BeginRequest` 可以在所有方法处理程序执行之前设置公共数据
    // 控制器可以像往常一样接受来自MVC应用程序路由的中间件
    //
    // 让我们看一个' ctx.Proceed '的例子:
    //
    // var authMiddleware = basicauth.New(basicauth.Config{
    // Users: map[string]string{
    // "admin": "password",
    // },
    // })
    //
    // func (c *UsersController) BeginRequest(ctx iris.Context) {
    // if !ctx.Proceed(authMiddleware) {
    // ctx.StopExecution()
    // }
    // }
    // 这个Get()方法将在与 `BeginRequest` 相同的处理程序中执行, 内部控制器检查 `ctx.StopExecution`
    // 如果 BeginRequest调用 `StopExecution` , 它不会被触发
    // func(c *UsersController) Get() []models.User {
    //    return c.Service.GetAll()
    //}
    // 另一种方法是 `!ctx.IsStopped()` 如果中间件在失败时使用' ctx.StopExecution() '
    Proceed(Handler) bool
    // HandlerName返回当前处理程序的名称, 有助于调试
    HandlerName() string
    // HandlerFileLine返回当前运行的处理程序的函数源文件和行信息
    // 调试的时候最常使用
    HandlerFileLine() (file string, line int)
    // RouteName返回运行此处理程序的路由名
    // 注意: 如果没找到处理程序会返回空
    RouteName() string
    // Next调用处理程序链中的所有下一个处理程序, 它应该在中间件中使用
    // 注意:自定义上下文应该覆盖这个方法, 以便能够传递自己的context.Context实现
    Next()
    // NextOr检查chain是否有下一个处理程序, 如果有, 则执行它
    // 否则, 它会根据给定的处理程序设置一个新的链赋值给这个上下文并执行它
    // 如果下一个处理器存在并执行, 则返回true, 否则返回false
    // 注意: 如果没有找到下一个处理程序, 并且处理程序缺失
    // 那么它发送一个状态Not Found(404)给客户端并停止执行
    NextOr(handlers ...Handler) bool
    // NextOrNotFound检查chain是否有下一个处理程序, 如果有, 则执行它
    // 否则它发送一个状态Not Found(404)给客户端并停止执行
    // 如果下一个处理器存在并执行, 则返回true, 否则返回false
    NextOrNotFound() bool
    // NextHandler从处理程序链返回(不执行)下一个处理程序
    // 如果需要执行返回的处理程序的下一个, 使用 `.skip()` 跳过这个处理程序
    NextHandler() Handler
    // Skip跳过/忽略处理程序链中的下一个处理程序, 它应该在中间件中使用
    Skip()
    // StopExecution停止此请求的处理程序链
    // 这表示任何接下来的 `Next` 调用都会被忽略, 结果是链中的下一个处理程序不会被触发
    StopExecution()
    // IsStopped报告上下文处理程序的当前位置是否为-1
    // (表示StopExecution()至少被调用一次)
    IsStopped() bool
    // StopWithStatus停止处理程序链并写入 "statusCode"
    // 如果状态代码是失败, 那么它还将触发指定的错误代码处理程序
    StopWithStatus(statusCode int)
    // StopWithText停止处理程序链并在 "plainText" 消息中写入 "statusCode"
    // 如果状态代码是失败, 那么它还将触发指定的错误代码处理程序
    StopWithText(statusCode int, plainText string)
    // StopWithError停止处理程序链并在 "err" 错误中写入 "statusCode"
    // 如果状态代码是失败, 那么它还将触发指定的错误代码处理程序
    StopWithError(statusCode int, err error)
    // StopWithJSON停止处理程序链, 写入状态代码, 发送一个JSON响应
     // 如果状态代码是失败, 那么它还将触发指定的错误代码处理程序
    StopWithJSON(statusCode int, jsonObject interface{})
    // StopWithProblem停止处理程序链, 写入状态代码
    // 并发送一个application/problem+json响应
    // 参见 `iris.NewProblem` 来正确构建一个 "problem" 值
    // 如果状态代码是失败, 那么它还将触发指定的错误代码处理程序
    StopWithProblem(statusCode int, problem Problem)
    // OnConnectionClose注册 "cb" 函数, 当底层连接消失时, 该函数将触发(在它自己的goroutine上, 不需要由end-dev注册goroutine)
    // 如果客户端在响应准备好之前断开连接, 这个机制可以用来取消服务器上的长时操作
    // 它取决于 `http#CloseNotify`
    // CloseNotify可能会等待通知直到请求, 正文已被完整读取
    // 主处理器返回后, 不保证通道接收到值
    // 最后, 它报告协议是否支持管道(禁用管道的HTTP/1.1不支持)
    // 如果输出值为false, 则"cb"不会被触发
    // 请注意, 你只能为整个请求处理程序链注册一个回调函数
    // 查看 ResponseWriter# closeotifier 了解更多
    OnConnectionClose(fnGoroutine func()) bool
    // OnClose使用 `Context#OnConnectionClose` 注册回调函数 "cb" 到标注连接关闭事件, 并在请求处理程序的最后使用 "ResponseWriter#SetBeforeFlush" 注册回调函数
    // 注意: 您只能为整个请求处理程序链注册一个回调函数
    // 注意 "cb" 只会被调用一次
    // 查看`Context#OnConnectionClose` 和 `ResponseWriter#SetBeforeFlush` 了解更多
    OnClose(cb func())

    //  +--------------------------------------+
    //  | 当前的 "user/request" 存储           |
    //  | 在处理程序之间共享信息 - Values()     |
    //  | 保存并获取命名路径参数 - Params()     |
    //  +--------------------------------------+

    // Params返回当前url的命名参数键值存储
    // 命名路径参数保存在这里
    // 这个存储, 作为整个上下文, 是每个请求的生存期
    Params() *RequestParams

    // Values返回当前的 "user" 存储
    // 命名路径参数和任何可选数据都可以保存在这里
    // 这个存储, 作为整个上下文, 是每个请求的生存期
    // 您可以使用此函数设置和获取本地值
    // 这些值可用于在处理程序和中间件之间共享信息
    Values() *memstore.Store

    //  +------------------------------------------------------------+
    //  | 路径, Host, 子域名, IP地址, 消息头, 定位 等等...    |
    //  +------------------------------------------------------------+

    // Method返回request.Method, 客户端对于服务端的http方法
    Method() string
    // Path返回完整的请求路径, 如果enablepathscape配置字段为true则转义
    Path() string
    // RequestPath基于 'escape' 返回完整的请求路径
    RequestPath(escape bool) string
    // Host返回当前url的host部分
    // 这个方法也用了 `Configuration.HostProxyHeaders` 
    Host() string
    // Subdomain返回此请求的子域名(如果有的话)
    // 注意: 这是一种快速方法, 但并不是所有情况都适用
    Subdomain() (subdomain string)
    // FindClosest返回一个 "n" 路径的列表
    // 该请求基于子域名和请求路径
    // 命令可能会改
    // 例子: https://github.com/kataras/iris/tree/master/_examples/routing/intelligence/manual
    FindClosest(n int) []string
    // IsWWW在子域名(如果有的话)为 "www" 时返回true 
    IsWWW() bool
    // FullRqeuestURI返回完整的uri, 包括方案、host和相对请求的路径/资源
    FullRequestURI() string
    // RemoteAddr尝试解析并返回真实客户端的请求IP
    //
    // 基于可以从 Configuration.RemoteAddrHeaders 中修改的合法消息头名称
    //
    // 如果基于这些头的解析失败, 那么它将返回请求的 `RemoteAddr` 字段
    // 该字段在HTTP处理程序之前由服务器填充
    //
    // 参见 `Configuration.RemoteAddrHeaders`,
    //      `Configuration.WithRemoteAddrHeader(...)`,
    //      `Configuration.WithoutRemoteAddrHeader(...)`
    //      `Configuration.RemoteAddrPrivateSubnets` 了解更多
    RemoteAddr() string
    // GetHeader根据请求头的名称返回请求头的值
    GetHeader(name string) string
    // GetDomain解析并返回服务器的域
    GetDomain() string
    // IsAjax在这个请求是一个 'ajax request'(XMLHttpRequest)时返回true
    //
    // 不可能100%知道请求是通过Ajax发出的
    // 您永远不应该相信来自客户端的数据, 通过干扰可以很容易地改动
    //
    // 注意: "X-Requested-With" 消息头能被任何客户端修改(因为"X-"),
    // 因此, 不要依赖于IsAjax来做真正重要的事情, 
    // 试着找到另一种方法来检测类型(比如内容类型)
    // 有许多博客描述了这些问题并提供了不同的解决方案, 
    // 这总是取决于您正在构建的应用程序, 
    // 这就是为什么“IsAjax”对于一般用途来说足够简单的原因
    //
    // 访问 https://developer.mozilla.org/en-US/docs/AJAX
    // 和 https://xhr.spec.whatwg.org/ 了解更多
    IsAjax() bool
    // IsMobile检查客户端是否使用移动设备(手机或平板电脑)与此服务器通信
    // 如果返回值为true, 表示http客户端使用移动设备与服务器通信, 否则为false
    // 请注意, 这将检查“User-Agent”请求头
    IsMobile() bool
    // IsScript报告客户端是否为脚本
    IsScript() bool
    // IsSSL报告客户端是否运行在HTTPS SSL下
    //
    // 参见 `IsHTTP2` 
    IsSSL() bool
    // IsHTTP2报告传入请求的协议版本是否为HTTP/2
    // 客户端要么用HTTP/1.1, 要么用HTTP/2
    //
    // 参见 `IsSSL`
    IsHTTP2() bool
    // IsGRPC报告请求是否来自gRPC客户端
    IsGRPC() bool
    // GetReferrer从 "Referer" (或"Referrer")消息头中提取并返回信息
    // 指定的url查询参数参见 https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy.
    GetReferrer() Referrer
    // SetLanguage强制设置i18n的语言, 可以在中间件内部使用
    // 最高优先于其他, 如果为空时, 它就将被忽略, 如果设置为 "default" 的静态字符串或默认语言的代码, 那么其他语言提取器将不会被调用, 默认语言将被设置
    //
    // 参见 `app.I18n.ExtractFunc` 一种更有组织的相同功能
    SetLanguage(langCode string)
    // GetLocale返回i18n中间件找到的当前请求的 `Locale`
    // 参见 `Tr`
    GetLocale() Locale
    // Tr基于带有可选参数的格式返回一个i18n本地化消息
    // 参见 `GetLocale` 
    // 例子: https://github.com/kataras/iris/tree/master/_examples/i18n
    Tr(format string, args ...interface{}) string
    // SetVersion强制设置与 "iris/versioning" 子包集成的API版本
    // 可被用于中间件中
    SetVersion(constraint string)
    //  +--------------------+
    //  | 消息头助手         |
    //  +--------------------+

    // Header给响应写入器添加一个消息头
    Header(name string, value string)

    // ContentType将响应写入器的头键 "Content-Type" 设置为 "cType"
    ContentType(cType string)
    // GetContentType返回响应写入器的头值 "Content-Type"
    // 它可以在之前的 "ContentType" 中设置
    GetContentType() string
    // GetContentType返回 "Content-Type" 的请求头值
    GetContentTypeRequested() string

    // GetContentLength返回 "Content-Length" 的请求头值
    // 如果找不到标头或其值不是有效数字, 则返回0
    GetContentLength() int64

    // StatusCode将状态码头设置为响应
    // 参见 `GetStatusCode`
    StatusCode(statusCode int)
    // GetStatusCode返回响应的当前状态码
    // 参见 `StatusCode` 
    GetStatusCode() int

    // AbsoluteURI解析 "s" 并返回它的绝对uri形式
    AbsoluteURI(s string) string
    // Redirect向客户端发送重定向响应到特定的url或相对路径
    // 接受两个个string形参和一个可选的int形参
    // 第一个参数是重定向的url
    // 第二个参数是应该发送的http状态
    // 默认为302 (StatusFound)
    // 你可以把它设置为301(永久重定向)
    // 如果是POST方法则设置303 (StatusSeeOther)
    // 如果需要的话也可设置StatusTemporaryRedirect(307)
    Redirect(urlToRedirect string, statusHeader ...int)
    //  +---------------------------+
    //  | 各种请求和Post数据         |
    //  +---------------------------+

    // URLParam当url参数存在时返回true, 否则返回false
    URLParamExists(name string) bool
    // URLParamDefault从请求中返回get参数
    // 如果没有找到, 则返回 "def"
    URLParamDefault(name string, def string) string
    // URLParam则返回请求的get参数(如果有的话)
    URLParam(name string) string
    // URLParamTrim返回url查询参数并删除后面的空格
    URLParamTrim(name string) string
    // URLParamEscape从请求中返回转义的url查询参数
    URLParamEscape(name string) string
    // URLParamInt从请求中以int形式返回url查询参数
    // 如果解析失败则返回-1和一个错误
    URLParamInt(name string) (int, error)
    // URLParamIntDefault从请求中以int形式返回url查询参数
    // 如果没有找到或解析失败, 则返回 "def"
    URLParamIntDefault(name string, def int) int
    // URLParamInt32Default从请求中以int32形式返回url查询参数,
    // 如果没有找到或解析失败, 则返回 "def"
    URLParamInt32Default(name string, def int32) int32
    // URLParamInt64从请求中以int64形式返回url查询参数
    // 如果解析失败则返回-1和一个错误
    URLParamInt64(name string) (int64, error)
    // URLParamInt64Default从请求中以int64形式返回url查询参数,
    // 如果没有找到或解析失败, 则返回 "def"
    URLParamInt64Default(name string, def int64) int64
    // URLParamFloat64从请求中以float64形式返回url查询参数
    // 如果解析失败则返回-1和一个错误
    URLParamFloat64(name string) (float64, error)
    // URLParamFloat64Default从请求中以float64形式返回url查询参数
    // 如果没有找到或解析失败, 则返回 "def"
    URLParamFloat64Default(name string, def float64) float64
    // URLParamBool从请求中以布尔值形式返回url查询参数
    // 如果解析失败则返回一个错误
    URLParamBool(name string) (bool, error)
    // URLParams返回GET查询参数的映, 如果有多个参数, 则用逗号分隔
    //如果没有找到, 它返回一个空的映射
    URLParams() map[string]string

    // FormValueDefault根据其 "name" 返回单个已解析的表单值
    // 包括url字段的查询参数和POST或PUT表单数据
    //
    // 如果没有找到, 则返回 "def"
    FormValueDefault(name string, def string) string
    // FormValue根据其 "name" 返回单个已解析的表单值,
    // 包括url字段的查询参数和POST或PUT表单数据
    FormValue(name string) string
    // FormValues返回解析后的表单数据
    // 包括URL字段的查询参数和POST或PUT表单数据
    // 默认形式的内存最大大小是32MB, 它可以通过 'iris#WithPostMaxMemory' 配置器在主配置传递 `app.Run` 的第二个参数更改
    // 注意: 需检查是否为nil
    FormValues() map[string][]string
    // PostValueDefault返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没有找到, 则返回 "def"
    PostValueDefault(name string, def string) string
    // PostValue返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    PostValue(name string) string
    // PostValueTrim返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据, 删去尾部空格
    PostValueTrim(name string) string
    // PostValueInt以int形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没找到, 则返回-1和一个不为nil的错误
    PostValueInt(name string) (int, error)
    // PostValueIntDefault以int形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没有找到, 则返回 "def" 或解析错误
    PostValueIntDefault(name string, def int) int
    // PostValueInt64以float64形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没找到, 则返回-1和一个不为nil的错误
    PostValueInt64(name string) (int64, error)
    // PostValueInt64Default以int64形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没有找到, 则返回 "def" 或解析错误
    PostValueInt64Default(name string, def int64) int64
    // PostValueFloat64以float64形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没找到, 则返回-1和一个不为nil的错误
    PostValueFloat64(name string) (float64, error)
    // PostValueFloat64Default以float64形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没有找到, 则返回 "def" 或解析错误
    PostValueFloat64Default(name string, def float64) float64
    // PostValueBool以bool形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 如果没找到或值为false, 则返回一个false, 否则返回true
    PostValueBool(name string) (bool, error)
    // PostValues以字符串切片形式返回POST、PATCH、PUT体参数中
    // 基于 "name" 的解析表单数据
    // 默认形式的内存最大大小是32MB, 它可以通过 " iris#WithPostMaxMemory" 
    // 配置器在主配置传递 `app.Run` 的第二个参数改变
    PostValues(name string) []string
    // FormFile返回从客户端接收到的第一个上传文件
    // 默认形式的内存最大大小是32MB, 它可以通过 " iris#WithPostMaxMemory" 
    // 配置器在主配置传递 `app.Run` 的第二个参数改变
    // 例子: https://github.com/kataras/iris/tree/master/_examples/file-server/upload-file
    FormFile(key string) (multipart.File, *multipart.FileHeader, error)
    // UploadFormFiles上传从客户端接收到的任何文件到系统物理位置 "destDirectory".
    
    // 第二个可选参数"before"给调用者修改*miltipart的机会, 文件头在保存到磁盘之前
    // 可以根据当前请求更改文件的名称, 文件头的所有选项都可以更改
    // 如果在将文件保存到磁盘之前不需要使用此功能, 您可以忽略它

    // 注意, 它不检查请求体是否为流

    // 如果由于操作系统的权限或 http.ErrMissingFile没有收到文件
    // 从而导致至少一个新文件无法创建, 则返回复制长度为int64和一个非nil错误

    // 如果你想接收文件并手动管理它们, 你可以使用 `context#FormFile` 来代替
    // 并创建一个适合你需要的复制函数, 下面是通用用法

    // 默认形式的内存最大大小是32MB, 它可以通过 " iris#WithPostMaxMemory" 
    // 配置器在主配置传递 `app.Run` 的第二个参数改变

    // 参见 `FormFile` 获取接收文件的更多控制

    // 例子: https://github.com/kataras/iris/tree/master/_examples/file-server/upload-files
    UploadFormFiles(destDirectory string, before ...func(Context, *multipart.FileHeader)) (n int64, err error)

    //  +--------------------+
    //  | 自定义HTTP错误     |
    //  +--------------------+

    // NotFound使用特定的自定义错误错误处理程序向客户端发出错误404
    // 注意: 如果你不希望下一个处理程序被执行, 你可能需要调用 ctx.StopExecution() 下一个处理程序将在iris上执行, 因为您可以替换错误代码并将其更改为更特定的错误代码, 即
    // users := app.Party("/users")
    // users.Done(func(ctx context.Context){ if ctx.StatusCode() == 400 { /*  custom error code for /users */ }})
    NotFound()

    //  +---------------------+
    //  | Body Readers        |
    //  +---------------------+

    // SetMaxRequestBodySize,在从客户端读取请求正文之前, 会对请求体大小设限制
    SetMaxRequestBodySize(limitOverBytes int64)
    // GetBody读取并返回请求主体
    // http请求读取器的默认行为消耗被读取的数据, 但是你可以通过传递 "WithoutBodyConsumptionOnUnmarshal" 这个iris选项来改变这个行为
    // 然而, 您在任何时候都可用 `ctx.Request().Body` 代替
    GetBody() ([]byte, error)
    // UnmarshalBody读取请求体并将其绑定到任何类型的值或指针
    // 使用例子: context.ReadJSON, context.ReadXML.
    //
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-custom-via-unmarshaler/main.go
    //
    // UnmarshalBody不检查gzip数据
    // 不要依赖传入服务器的压缩数据, 主要原因是: https://en.wikipedia.org/wiki/Zip_bomb
    // 然而, 你仍然可以自由地阅读 `ctx.Request().Body io.Reader` 
    UnmarshalBody(outPtr interface{}, unmarshaler Unmarshaler) error
    // ReadJSON从请求体中读取JSON, 并将其绑定到任何JSON有效类型的值指针上
    //
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-json/main.go
    ReadJSON(jsonObjectPtr interface{}) error
    // ReadXML从请求体中读取XML, 并将其绑定到任何XML有效类型的值指针上
    //
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-xml/main.go
    ReadXML(xmlObjectPtr interface{}) error
    // ReadYAML从请求体中读取YAML, 并将其绑定到任何 `outPtr` 值上
    //
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-yaml/main.go
    ReadYAML(outPtr interface{}) error
    // ReadForm将表单的请求体绑定到 "formObject"
    // 它支持任何类型的类型, 包括自定义的结构体
    // 如果请求数据为空, 则返回空值
    // struct字段标签是 "form"
    // 请注意, 如果 `Configuration.FireEmptyFormError` 为空表单数据, 它将返回nil error
    // 在这种情况下, 调用者应该检查指向的指针, 查看是否被绑定
    // 如果客户端发送一个未知的字段, 这个方法将返回一个错误
    // 为了忽略该错误, 请使用 `err != nil && !iris.IsErrPath(err)`
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-form/main.go
    ReadForm(formObject interface{}) error
    // ReadQuery绑定url查询到 "ptr" , struct字段标签是"url"
    // 如果客户端发送一个未知的字段, 这个方法将返回一个错误, 
    // 为了忽略该错误, 请使用 `err != nil && !iris.IsErrPath(err)`
    // 例子: https://github.com/kataras/iris/blob/master/_examples/request-body/read-query/main.go
    ReadQuery(ptr interface{}) error
    // ReadProtobuf将正文绑定到 proto消息的 "ptr" , 并返回任何错误
    ReadProtobuf(ptr proto.Message) error
    // ReadMsgPack将msgpack格式的请求正文绑定到 "ptr", 并返回任何错误
    ReadMsgPack(ptr interface{}) error
    // ReadBody根据HTTP方法和请求的内容类型将请求正文绑定到 "ptr"
    // 如果一个GET方法请求, 那么它从一个表单(或URL查询)中读取, 否则
    // 它将尝试匹配(取决于请求的内容类型)数据格式, 例如:
    // JSON, Protobuf, MsgPack, XML, YAML, MultipartForm并将结果绑定到 "ptr"
    ReadBody(ptr interface{}) error

    //  +--------------------+
    //  | 体(未处理)写入器    |
    //  +--------------------+

    // Write将数据作为HTTP应答的一部分写入连接

    // 如果WriteHeader还没有被调用, 
    // Write会在写入数据之前调用 WriteHeader(http.StatusOK)
    // 如果消息头不包含Content-Type行, Write会在将初始传递给 DetectContentType 的512字节
    // 写入数据结果中添加Content-Type行

    // 根据HTTP协议版本和客户端, 调用Write或WriteHeader可能会阻止将来读取 Request.Body
    // 对于HTTP/1.x请求, 处理程序应该在写入响应之前读取任何需要的请求体数据
    // 一旦头文件被刷新(由于显式刷新器)Flush调用或写入足够的数据以触发Flush)时
    // 请求体可能不可用对于HTTP/2请求, Go HTTP服务器允许处理程序继续读取请求体
    // 同时并发地写入响应
    // 然而, 并非所有HTTP/2客户端都支持这种行为
    // 如果可能的话, 处理程序应该先读再写, 以最大化兼容性
    Write(body []byte) (int, error)
    // Writef根据格式说明符格式化并写入响应
    // 返回写入的字节数和遇到的任何写入错误
    Writef(format string, args ...interface{}) (int, error)
    // WriteString将一个简单的字符串写入响应
    // 返回写入的字节数和遇到的任何写入错误
    WriteString(body string) (int, error)

    // SetLastModified基于 "modtime" 输入来设置 "Last-Modified"
    //如果 "modtime" 是0, 那么它什么也不做
    
    // 它主要在core/router和context包内部
    
    // 注意: modtime. UTC() 不仅仅被modtime使用
    // 所以你不需要知道它的内部结构
    SetLastModified(modtime time.Time)
    // CheckIfModifiedSince检查 "modtime" 的响应是否被修改
    // 注意, 它与服务端缓存无关
    // 它通过检查由客户端发送或之前的服务器响应头(例如:WriteWithExpiration或HandleDir或Favicon等)
    // 发送的"If-Modified-Since" 请求头
    // 是否有效并且在“modtime”之前来实现检查过程

    // 检查 !modtime && err == nil 是必要的, 以确保它未被修改, 因为虽然它可能返回false,  
    // 但没有有机会检查客户端(请求)头由于一些错误, 比如HTTP方法不是 "GET" 或 "HEAD" 
    // 或者 "modtime"是0, 或者从消息头中解析时间失败
    
    // 它通常在内部使用, 例如: `context# WriteWithExpiration` 
    // 参见 `ErrPreconditionFailed`

    // 注意: modtime. UTC() 不仅仅被modtime使用
    // 所以你不需要知道它的内部结构
    CheckIfModifiedSince(modtime time.Time) (bool, error)
    // WriteNotModified发送一个304 "未修改" 状态码到客户端
    // 它确保内容类型, 内容长度头和任何 "ETag" 在响应发送之前被删除
    // 它主要在core/router/fs.go和上下文方法中使用
    WriteNotModified()
    // WriteWithExpiration类似于 `Write` , 
    // 但它会根据输入参数modtime检查资源是否被修改
    //否则发送一个304状态码, 以便让客户端渲染缓存的内容
    WriteWithExpiration(body []byte, modtime time.Time) (int, error)
    // StreamWriter注册给定的流编写器以填充响应体
    // 禁止写入器访问上下文和/或其成员
    // 这个函数可以在以下情况下使用:
    // *如果响应体太大(大于iris.LimitRequestBodySize(如果设置的话))。
    // *如果响应体是从缓慢的外部资源流入(stream)
    // *如果响应体必须以块的形式流入客户端
    // (又名为 `http server push`)
    // 接收响应写入器的函数
    // 并在应该停止写入时返回false, 否则返回true以便继续写入
    StreamWriter(writer func(w io.Writer) bool)

    //  +-------------------+
    //  | 带压缩的体写入器   |
    //  +-------------------+
    // ClientSupportsGzip当客户端支持gzip压缩时返回true
    ClientSupportsGzip() bool
    // WriteGzip接收字节, 这些字节被压缩为gzip格式并发送给客户端
    // 返回已写入的字节数和一个错误(如果客户端不支持gzip压缩)
    // 你可以在同一个处理程序中重用这个函数来多次写入更多的数据, 不会有任何问题
    WriteGzip(b []byte) (int, error)
    // TryWriteGzip接收字节, 这些字节被压缩为gzip格式并发送给客户端
    // 如果客户端不支持gzip, 那么内容就按原样写入, 不压缩
    TryWriteGzip(b []byte) (int, error)
    // GzipResponseWriter将当前的响应写入器转换为响应写入器
    // 当新的写入器的 .write被调用时, 它将数据压缩为gzip并将其写入客户端
    // 也可以通过它的 .disable和 .resetbody来禁用, 以回滚到通常的响应写入器
    GzipResponseWriter() *GzipResponseWriter
    // Gzip可以启用或禁用(如果之前启用)gzip响应写入器
    // 如果客户端支持gzip压缩, 那么下面的响应数据将作为压缩的gzip数据发送到客户端
    Gzip(enable bool)
    // GzipReader接受一个布尔值, 如果设置为true, 它将使用gzip读入器包装请求正文读入器(在读取时解压缩请求数据)
    //如果 "enable" 输入参数为false, 则请求体将重置为默认值
    // 当传入的请求数据被gzip压缩时很有用
    // `ctx.GetBody/ReadXXX/UnmarshalBody` 方法的所有将来调用将遵循此选项
    //
    // 使用:
    // app.Use(func(ctx iris.Context){
    // ctx.GzipReader(true)
    // ctx.Next()
    // })
    //
    // 如果客户端请求的主体不是gzip压缩的
    // 那么它返回 `ErrGzipNotSupported` 错误, 可以安全地忽略
    // 参见 `GzipReader` 包级中间件获取更多
    GzipReader(enable bool) error

    //  +-----------------------------+
    //  | 丰富的正文内容写入器/渲染器  |
    //  +-----------------------------+

    // ViewLayout当 .view在相同的请求中被调用时设置"layout"选项
    // 当需要根据链中的前一个处理程序设置或/和更改布局时非常有用
    // 注意: 'layoutTmplFile' 参数可以设置为iris.NoLayout || view.NoLayout
    // 以便禁用特定视图渲染操作的布局, 它禁用引擎配置的布局属性
    
    // 参见 .ViewData 和 .View 获取更多
    
    // 例子: https://github.com/kataras/iris/tree/master/_examples/view/context-view-data/
    ViewLayout(layoutTmplFile string)
    // ViewData保存一个或多个键值对, 以便在随后同一个请求中 .view被调用时传递它们
    // 当需要从链中先前的处理程序中设置或/和更改模板数据时很有用
    // 如果 .view的 "binding" 参数不是nil并且它不是map类型, 那么这些数据会被忽略 
    // binding有优先级, 所以主路由的处理器仍然可以决定
    // 如果绑定是一个map或context然后将这些数据添加到视图数据并传递给模板
    // 在.View之后, 数据不会被销毁, 以便在需要的时候被重用(其他所有的请求都一样), 为了清除视图数据, 开发人员可以调用:
    // ctx.Set (ctx.Application().ConfigurationReadOnly().GetViewDataContextKey(), nil)
    //如果 'key' 为空, 那么该值就会被添加为(struct或map), 而开发者无法添加其他值
    // 参见 .ViewLayout 和 .View 获取更多
    // 例子: https://github.com/kataras/iris/tree/master/_examples/view/context-view-data/
    ViewData(key string, value interface{})
    // GetViewData返回由 `context#ViewData` 注册的值。
    // 返回值是 `map[string]interface{}` , 这意味着
    // 如果一个自定义结构体注册到ViewData, 那么这个函数
    // 将尝试将它解析为map, 如果失败则返回值为nil
    // 如果通过 `ViewData` 注册了不同类型的值或没有数据,那么检查nil总是一个很好的做法
    // 类似于 `viewData:= ctx.Values().Get("iris.viewData")` 或
    // `viewData := ctx.Values().Get (ctx.Application(). ConfigurationReadonly(). Getviewdatacontextkey()) '
    GetViewData() map[string]interface{}
    // View基于注册的视图引擎渲染模板
    // 第一个参数接受文件名, 相对于视图引擎的目录和扩展名
    // 如果目录是 "./templates/users/index.html"
    // 然后你传递 "users/index.html" 作为filename参数
    // 第二个可选参数可以接收一个 "view model"
    // 如果它不是nil, 它将被绑定到视图模板上 
    // 否则它将检查之前存储在ViewData中的视图数据
    // 即使为相同的请求而存储在任何以前的处理器(中间件)

    // 参见 .ViewData` and .ViewLayout 获取更多
    // 例子: https://github.com/kataras/iris/tree/master/_examples/view
    View(filename string, optionalViewModel ...interface{}) error

    // Binary以二进制数据的形式写出原始字节
    Binary(data []byte) (int, error)
    // Text以纯文本的形式写出一个字符串
    Text(format string, args ...interface{}) (int, error)
    // HTML以text/html形式写出一个字符串
    HTML(format string, args ...interface{}) (int, error)
    // JSON编码给定的接口对象并写入JSON响应
    JSON(v interface{}, options ...JSON) (int, error)
    // JSONP编码给定的接口对象并写入JSON响应
    JSONP(v interface{}, options ...JSONP) (int, error)
    // XML编码给定的接口对象并写入XML响应
    // 要将映射呈现为XML, 请参见 `XMLMap` 包级函数
    XML(v interface{}, options ...XML) (int, error)
    // Problem写入一个JSON或XML问题响应
    // Problem字段的顺序并不总是相同的
    //
    // 和 `Context.JSON` 的行为完全一样
    // 但是默认 ProblemOptions.JSON缩进为 " " , 响应内容类型为 "application/problem+json"
    //
    //使用 options.RenderXML 和 XML 字段改变这个行为
    // 并且发送响应内容类型为 "application/problem+xml" 代替
    //
    // 阅读更多: https://github.com/kataras/iris/wiki/Routing-error-handlers
    Problem(v interface{}, opts ...ProblemOptions) (int, error)
    // Markdown将markdown解析为html, 并将结果渲染给客户端
    Markdown(markdownB []byte, options ...Markdown) (int, error)
    // YAML使用yaml解析器解析 "v" , 并将结果渲染给客户端
    YAML(v interface{}) (int, error)
    // Protobuf解析proto消息的 "v" , 并将其结果渲染给客户端
    Protobuf(v proto.Message) (int, error)
    // MsgPack解析msgpack格式的 "v" , 并将其结果呈现给客户端
    MsgPack(v interface{}) (int, error)
    //  +-----------------------------------------------------------------------+
    //  | 内容协商                                                              |
    //  | https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation |                                       |
    //  +-----------------------------------------------------------------------+
    // Negotiation创建一次并返回协商构建器, 以构建特定mime类型(们)和字符集(们)的服务器端可用内容
    // 参见 `Negotiate` 方法获取更多 
    Negotiation() *NegotiationBuilder
    // Negotiate用于为同一URI上资源的不同表示提供服务
    // "v"可以是一个单一的' N '结构体值
    // "v"可以是' ContentSelector '接口的任意值
    // "v"可以是' ContentNegotiator '界面的任意值
    // "v"可以是struct(JSON, JSONP, XML, YAML)或
    // 任何匹配mime类型的字符串(TEXT, HTML)或[]字节(Markdown, 二进制)或[]字节
    // 如果 "v" 为nil, 则使用 `context.Negotiate()` 构建器的内容代替
    // 否则 "v" 将覆盖构建器的内容(服务器mime类型仍然通过其注册的、支持的mime列表检索)
    
    // 通过 `Negotiation().JSON().XML().HTML()…` 设置mime类型优先级
    // 通过 `Negotiation().Charset(…)` 设置字符集优先级
    // 通过 `Negotiation().Encoding(…)` 设置编码算法优先级

    // 修改 `Negotiation().Accept./Override()/ .XML().JSON().Charset(…).Encoding(…)…` 接收的内容
    // 当不匹配mime类型(们)时, 返回 `ErrContentNotSupported`

    // 资源:
    // https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation
    // https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept
    // https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Charset
    // https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding
    //
    // 支持上述没有quality值
    // 阅读更多: https://github.com/kataras/iris/wiki/Content-negotiation
    Negotiate(v interface{}) (int, error)

    //  +-----------+
    //  | 服务文件  |
    //  +-----------+

    // ServeContent使用提供的ReadSeeker中的内容回复请求
    // 与 io.Copy 相比, servcontent的主要优点是正确地处理范围请求
    // 设置MIME类型, 并处理If-Match、If-Unmodified-Since、If-None-Match、If-Modified-Since和If-Range请求

    // 如果响应的Content-Type头没有设置, 则ServContent首先尝试从name的文件扩展名推断类型
    // 该名称未被使用; 特别地, 它可以是空的, 并且不会在响应中发送
    // 如果modtime不是0时间或Unix时间, ServContent在响应的Last-Modified头中包含它
    // 如果请求包含If-modified-since头, ServeContent使用modtime来决定是否需要发送内容
    // 内容的Seek方法必须工作:ServContent使用一个到内容末尾的Seek来确定它的大小
    // 如果调用者设置了按照RFC 7232第2.3节格式化的 w's ETag头, ServContent将使用它来处理使用If-Match、If- None-Match或If-Range请求

    // 注意: *os.File实现 io.ReadSeeker 接口

    // 注意: gzip压缩可以通过 `ctx.Gzip(true)` 或 `app.Use(iris.Gzip)` 注册 
    ServeContent(content io.ReadSeeker, filename string, modtime time.Time)
    // ServeContentWithRate和 `ServeContent` 一样但是它会降低阅读速度
    // 并且通过写 "content" 给客户端
    ServeContentWithRate(content io.ReadSeeker, filename string, modtime time.Time, limit float64, burst int)
    // ServeFile用命名文件或目录的内容响应请求
  
    // 如果提供的文件或目录名是一个相对路径, 它将相对于当前目录进行解释
    // 并可能上升到父目录。如果提供的名称是根据用户输入构造的
    // 那么在调用 `ServeFile` 之前应该对其进行清理
    
    // 当你想要提供css和javascript文件等资源时, 使用它
    // 如果客户端确认并保存文件, 使用 `SendFile` 代替
    // 注意: gzip压缩可以通过 `ctx.Gzip(true)`或 `app.Use(iris.Gzip)` 注册
    ServeFile(filename string) error
    // ServeFileWithRate和 `ServeFile` 一样但是它会降低阅读速度
    // 并且通过写文件给客户端
    ServeFileWithRate(filename string, limit float64, burst int) error
    // SendFile发送一个附件形式的文件, 该文件从客户端下载并在本地保存
    // 请注意gzip压缩可以通过 `ctx.Gzip(true)` 或 `app.Use(iris.Gzip)` 注册
    // 如果文件应该作为页面资源使用, 则使用 `ServeFile`
    SendFile(filename string, destinationName string) error
    // SendFileWithRate和 `SendFile` 一样但是它会降低阅读速度
    // 并且通过写文件给客户端
    SendFileWithRate(src, destName string, limit float64, burst int) error

    //  +-----------+
    //  | Cookies   |
    //  +-----------+

    // AddCookieOptions为SetCookie添加cookie选项
    // 在链中的下一个处理程序发送或接收cookie之前
    // 当前请求的 `SetCookieKV, UpsertCookie` 和 `RemoveCookie` 方法,
    // 可以从中间件调用

    // 参见 `ClearCookieOptions` 了解更多
    
    // 可用的内置Cookie选项有:
    //  * CookieAllowReclaim
    //  * CookieAllowSubdomains
    //  * CookieSecure
    //  * CookieHTTPOnly
    //  * CookieSameSite
    //  * CookiePath
    //  * CookieCleanPath
    //  * CookieExpires
    //  * CookieEncoding
    //
    // 例子: https://github.com/kataras/iris/tree/master/_examples/cookies/securecookie
    AddCookieOptions(options ...CookieOption)
    // ClearCookieOptions清除以前注册的cookie选项
    // 参见 `AddCookieOptions` 了解更多
    ClearCookieOptions()
    // SetCookie增加一条cookie
    // 不需要使用 "option", 它们可用于修改 "cookie"
    // 例子: https://github.com/kataras/iris/tree/master/_examples/cookies/basic
    SetCookie(cookie *http.Cookie, options ...CookieOption)
    // UpsertCookie像 `SetCookie` 那样在响应中添加一条cookie
    // 但如果先前设置了 `SetCookie`调用, 它也会执行一个替换cookie的操作
    // 它会报告cookie是新的(true)还是已更新的(false)
    UpsertCookie(cookie *http.Cookie, options ...CookieOption) bool
    // SetCookieKV添加一条cookie, 需要名称(字符串)和值(字符串)
    // 默认情况下, 它的过期时间为2小时, 它被添加到根路径, 使用 `CookieExpires` 和 `CookiePath` 修改它们
    // 另外: ctx.SetCookie(&http.Cookie{...})
    
    // 如果你想设置自定义路径:
    // ctx.SetCookieKV(name, value, iris.CookiePath("/custom/path/cookie/will/be/stored"))
    
    // 如果您希望仅对当前请求路径可见:
    // ctx.SetCookieKV(name, value, iris.CookieCleanPath/iris.CookiePath(""))
    // 更多:
    //               iris.CookieExpires(time.Duration)
    //               iris.CookieHTTPOnly(false)
    
    // 例子: https://github.com/kataras/iris/tree/master/_examples/cookies/basic
    SetCookieKV(name, value string, options ...CookieOption)
    // GetCookie根据名称返回cookie的值
    //如果没有找到, 返回空字符串

    // 如果你想要更多的值, 那么:
    // cookie, err := ctx.Request().Cookie("name")
    // 例子: https://github.com/kataras/iris/tree/master/_examples/cookies/basic
    GetCookie(name string, options ...CookieOption) string
    // RemoveCookie通过名称和path = "/"删除cookie
    // 小贴士: 将cookie的路径更改为当前的路径: RemoveCookie("name", iris.CookieCleanPath)
    // 例子: https://github.com/kataras/iris/tree/master/_examples/cookies/basic
    RemoveCookie(name string, options ...CookieOption)
    // VisitAllCookies接受一个访问函数, 该函数根据每个(请求的)cookie的名称和值调用
    VisitAllCookies(visitor func(name string, value string))
    // MaxAge返回 "cache-control" 请求头的秒值为int64
    // 如果头文件没有找到或者解析失败, 那么它返回-1
    MaxAge() int64
    //  +-------------------------+
    //  | 高级: 响应记录器和事务   |
    //  +-------------------------+

    // Record将上下文的基本和直接的responseWriter转换为ResponseRecorder, 
    // 它可以用于重置主体, 重置消息头, 获取主体, 在任何时候获取和设置状态代码等等
    Record()
    // Recorder返回上下文的ResponseRecorder
    // 如果没有记录, 它将开始记录并返回新上下文的ResponseRecorder
    Recorder() *ResponseRecorder
    // IsRecording在响应写入器记录状态码、正文、头文件等时, 返回响应记录器和true
    // 否则返回nil和false
    IsRecording() (*ResponseRecorder, bool)
    // BeginTransaction启动一个限定范围的事务
    // 你可以搜索关于商业事务如何工作的第三方文章或书籍(这很简单, 尤其是在这里)

    // 注意: 这是唯一的和最新的
    // (在Golang的这个主题上, 迄今为止, 关于大多数iris特性我从来没有看到任何其他例子或代码…)

    // 它不覆盖所有的路径, 例如数据库, 您使用数据库连接这应该由库管理, 这个事务范围只用于上下文的响应
    // 事务也有自己的中间件生态系统, 参见iris.go:UseTransaction
    // 参见 https://github.com/kataras/iris/tree/master/_examples/ 获取更多
    BeginTransaction(pipe func(t *Transaction))
    // SkipTransactions如果被调用, 则跳过其余事务, 如果在第一个事务之前调用, 则跳过所有事务
    SkipTransactions()
    // TransactionsSkipped如果事务被跳过或完全取消, 则返回true
    TransactionsSkipped() bool
    // Exec根据这个上下文调用 `context/Application# ServeCtx`  
    // 但与用户请求的方法和路径相同, 但它不是
    // 脱机意味着路由注册到iris, 具有普通路由的所有特性, 但不能通过浏览获得
    // 它的处理程序只有在其他处理程序的上下文调用它们时才执行它可以验证路径, 
    // 具有会话、路径参数等
    // 您可以通过app.GetRoute(" theoutename ")找到路由
    // app.Get("/mypath",  handler)("theRouteName")将为路由设置一个名称, 
    // 并返回它的 RouteInfo实例以供进一步使用
    // 它不会改变全局状态, 如果路由是 "脱机", 它仍然是脱机
    // app.None(...) and app.GetRoutes().Offline(route)/.Online(route, method)
    //
    // 例子: https://github.com/kataras/iris/tree/master/_examples/routing/route-state
    //
    // 用户可以简单地使用 rec := ctx.Recorder(); rec.Body()/rec.StatusCode()/rec.Header() 获取响应
    //
    // Context的值和会话会被保留, 以便能够通过结果路由进行通信
    //
    // 它适用于极端的用例, 99%的情况下都不会对你有用
    Exec(method, path string)

    // RouteExists报告特定路由是否存在
    // 如果不是在根域内的话, 它将从上下文主机的当前子域进行搜索
    RouteExists(method, path string) bool

    // ReflectValue缓存并返回一个 []reflect.Value{reflect.ValueOf(ctx)}
    // 它只是一个维护上下文内部变量的助手
    ReflectValue() []reflect.Value
    // Controller返回执行此处理程序的自定义控制器的反射值
    // 如果处理程序不是从控制器中执行的, 返回一个 Kind() == reflect.Invalid
    Controller() reflect.Value
    // RegisterDependency在服务时段为链中的下一个处理程序注册一个结构依赖项
    // 每种类型一个值
    // 注意: 强烈建议在服务器运行之前注册依赖项
    // 通过APIContainer(app.ConfigureContainer)或MVC(mvc.New)来降低性能成本
    // 参见 `UnregisterDependency` 获取更多
    RegisterDependency(v interface{})
    // UnregisterDependency根据其类型移除依赖项
    // 报告与该类型的依赖是否被成功找到并删除
    // 参见 `RegisterDependency` 获取更多
    UnregisterDependency(typ reflect.Type) bool
    // Application返回属于此上下文的iris应用实例
    // 值得注意的是, 这个函数返回一个应用程序的接口, 
    // 其中包含在服务时段可以安全执行的方法
    // 为了开发者的安全, 在这里不可以获得完整的应用字段和方法
    Application() Application
    // SetID设置一个ID, 一些值, 来请求上下文
    // 如果可能, "id" 应该实现 `String() string` 方法
    // 所以它可以被渲染在 `Context.String` 方法上
    // 参见 `GetID` and `middleware/requestid` 获取更多
    SetID(id interface{})
    // GetID返回请求的上下文ID.
    // 如果不是由先前的 `SetID` 调用给出, 它返回nil
    // 参见 `middleware/requestid` 获取更多
    GetID() interface{}
    // String返回此请求的字符串表示形式
    // 它返回由 `SetID` 调用给出的上下文ID, 然后是客户端的IP和 method:uri
    String() string
}
```
