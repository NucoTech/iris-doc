# URL query参数(URL query parameters)

Iris的Context有两种方法返回 `net/http` 标准 `http.ResponseWriter` 和 `http.Request`值, 我们在前面的章节已经提到过

- Context.Request()
- Context.ResponseWriter()

然而, 除了Iris Context提供的独特的Iris特性和辅助功能, 为了方便开发, 我们还提供了一些现有的 `net/http` 功能的装饰器

这有处理URL query字符串的完整方法列表可以帮您:

```go
// 如果url parameter存在, URLParam返回true, 否则返回false 
URLParamExists(name string) bool
// URLParamDefault返回请求中的获得的参数
// 如果没找到参数返回 "def"
URLParamDefault(name string, def string) string
// URLParam返回请求中的任何获得的参数
URLParam(name string) string
// URLParamTrim返回请求中删除url query参数的尾部空格
URLParamTrim(name string) string
// URLParamEscape返回请求中转义url query参数.
URLParamEscape(name string) string
// URLParamInt返回请求中的url query参数的int形式, 如果解析失败会返回-1和一个错误
URLParamInt(name string) (int, error)
// URLParamIntDefault返回请求中的url query参数的int形式, 如果没有找到或解析失败会返回 "def"
URLParamIntDefault(name string, def int) int
// URLParamInt32Default返回请求中的url query参数的int32形式, 如果没有找到或解析失败会返回 "def"
URLParamInt32Default(name string, def int32) int32
// URLParamInt64返回请求中的url query参数的int64形式, 如果解析失败会返回-1和一个错误
URLParamInt64(name string) (int64, error)
// URLParamInt64Default返回请求中的url query参数的int64形式, 如果没有找到或解析失败会返回 "def"
URLParamInt64Default(name string, def int64) int64
// URLParamFloat64返回请求中的url query参数的float64形式, 如果解析失败会返回-1和一个错误
URLParamFloat64(name string) (float64, error)
// URLParamFloat64Default返回请求中的url query参数的float64形式, 如果没有找到或解析失败会返回 "def"
URLParamFloat64Default(name string, def float64) float64
// URLParamBool返回请求中的url query参数的bool形式, 如果没有找到或解析失败会返回一个错误
URLParamBool(name string) (bool, error)
// URLParams返回GET查询参数的映射, 如果有多个参数, 则用逗号分隔, 如果没有找到, 它返回一个空的映射
URLParams() map[string]string
```

query字符串参数使用现有的底层请求对象进行解析, 请求响应url匹配类似: */welcome?firstname=Jane&lastname=Doe*:

- `ctx.URLParam("lastname") == ctx.Request().URL.Query().Get("lastname")`

代码示例:

```go
app.Get("/welcome", func(ctx iris.Context) {
        firstname := ctx.URLParamDefault("firstname", "Guest")
        lastname := ctx.URLParam("lastname") 

        ctx.Writef("Hello %s %s", firstname, lastname)
    })
```
