# Cookies

Cookies可以通过Context的请求实例访问, `ctx.Request()` 返回一个 `net/http#Request`

然而, Iris的 `Context` 提供了一些帮助, 方便访问最常见的Cookie, 而如果您只是使用请求的cookie方法, 则不需要任何自定义的额外代码

## 设置一条Cookie

`SetCookie` 和 `UpsertCookie` 方法添加或设置(如果允许还可删除)一条cookie

"options" 的最后一个参数是可选的, `CookieOption` 可用来修改 "cookie" , 稍后您需要看到可用的选项, 可以根据您的web应用需求添加自定义选项, 这有助于避免您在代码库中的重复

在我们继续之前, `Context` 也有一个 `SetSameSite(http.SameSite)` 方法, 它为所有即将发送的cookie设置 ["SameSite"](https://web.dev/samesite-cookies-explained/) cookie字段

```go
SetCookie(cookie *http.Cookie, options ...CookieOption)
```

如果您愿意, 您也可以使用 `SetCookieKV` 方法, 它不需要导入 `net/http` 包

```go
SetCookieKV(name, value string, options ...CookieOption)
```

请注意, `SetCookieKV` 设置的默认过期时间是365天, 您也可以使用 `CookieExpires` 这个Cookie选项(见下文)或者全局设置 `kataras/iris/Context.SetCookieKVExpiration` 包层的变量

`CookieOption` 只为 `func(*http.Cookie)` 而服务

### SetPath

```go
CookiePath(path string) CookieOption
```

### Set Expiration

```go
iris.CookieExpires(durFromNow time.Duration) CookieOption
```

### HttpOnly

```go
iris.CookieHTTPOnly(httpOnly bool) CookieOption
```

对于 `RemoveCookie` 和 `SetCookieKV` , HttpOnly 字段默认为true

我们再来讨论一下cookie的编码问题

### 编码(Encode)

在添加cookie时提供编码的功能

接收一个 `CoolieEncoder` 并且设置cookie的值设置为编码后的值

它的使用者是 `SetCookie` 和 `SetCookieKV`

```go
iris.CookieEncode(encode CookieEncoder) CookieOption
```

### 解码(Decode)

在检索cookie时提供解码功能

接收一个 `CookieDecoder` 并且在返回给 `GetCookie` 前设置cookie的值为解码后的值

它的使用者是  `GetCookie`

```go
iris.CookieDecode(decode CookieDecoder) CookieOption
```

其中, `CookieEncoder` 可以这样描述:

CookieEncoder应编码cookie的值

- 应接收cookie的名称作为它的第一个参数
- cookie的值的指针作为第二个参数
- 如果编码失败, 应返回一个已编码的值或一个空值
- 如果编码失败, 应返回一个错误

```go
CookieEncoder func(cookieName string, value interface{}) (string, error)
```

`CookieDecoder` 这样描述:

CookieDecoder应解码cookie的值

- 应接收cookie的名称作为它的第一个参数
- cookie解码后的值作为第二个参数, 解码后值的指针作为第三个参数
- 如果解码失败, 应返回一个已解码的值或一个空值
- 如果解码失败, 应返回一个错误

```go
CookieDecoder func(cookieName string, cookieValue string, v interface{}) error
```

错误不会被打印, 所以您必须心里有数, 记住: 如果您使用AES, 它只支持16、24或32字节的秘钥

您要么提供准确的数量, 要么从输入的内容中派生键

## 获得一条Cookie

`GetCookie` 方法按cookie的名称返回它的值, 未找到则返回空的字符串

```go
GetCookie(name string, options ...CookieOption) string
```

如果您想要更多的值那就用这个代替:

```go
cookie, err := ctx.Request().Cookie("name")
```

## 获得所有的Cookie

`ctx.Request().Cookies()` 方法返回所有可用请求cookie的切片, 有时候您想要修改它们或者对它们中的每一个执行一个操作

```go
VisitAllCookies(visitor func(name string, value string))
```

## 删除一条Cookie

`RemoveCookie` 方法可以根据cookie的名字和形如 "/xxx" 的地址, 根路径来删除它

小贴士: 通过提供 `iris.CookieCleanPath` Cookie选项可以更改cookie的路径为当前路径, 比如 `RemoveCookie("nname", iris.CookieCleanPath)`

而且注意, cookie的 `HttpOnly` 参数的默认选项是true, 它根据web标准执行删除cookie

```go
RemoveCookie(name string, options ...CookieOption)
```

[戳我](https://github.com/kataras/iris/wiki/Sessions)学习HTTP Session
