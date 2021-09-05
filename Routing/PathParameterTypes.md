# 路径参数类型(Routing path parameter types)

Iris拥有您所见过的最简单和最强大的路由流程。

Iris有自己的路由路径语法、解析和求值的interpeter(是的, 就像编程语言一样!)

它很快, 如何?它会计算自己的需求, 如果不需要任何特殊的regexp, 那么它只使用低级路径语法注册路由, 否则它会预编译regexp并添加必要的中间件。这意味着与其他路由器或web框架相比, 您的性能成本为零。

## 参数

路径参数的名称应该只包含字母。不允许使用 "_" 之类的数字或符号

不要混淆 `ctx.Params()` 和 `ctx.Values()`

- 路径参数的值可以从 `ctx.Params()` 中获取
- 用于处理程序和中间件之间通信的上下文本地存储可以存储到 `ctx.Values()` 中

可以在下表中找到内置的可用参数类型

| 参数类型 | Go类型 | 范围 | 检索辅助 |
| --- | --- | --- | --- |
|  `:string` | string | 任意(单路径段) | `Params().Get` |
| `:int` | int | -9223372036854775808 to 9223372036854775807 (x64) 或 -2147483648 到 2147483647 (x32), 取决于host arch | `Params().GetInt` |
| `:int8` | int8 | -128 到 127 | `Params().GetInt8` |
| `:int16` | int16 | -32768 到 32767 | `Params().GetInt16` |
| `:int32` | int32 | -2147483648 到 2147483647 | `Params().GetInt32` |
| `:int64` | int64 | -9223372036854775808 到 9223372036854775807 | `Params().GetInt64` |
| `:uint` | uint | 0 到 18446744073709551615 (x64) 或 0 到 4294967295 (x32), 取决于host arch | `Params().GetUint` |
| `:uint8` | uint8 | 0 到 255 | `Params().GetUint8` |
| `:uint16` | uint16 | 0 到 65535 | `Params().GetUint16` |
| `:uint32` | uint32 | 0 到 4294967295 | `Params().GetUint32` |
| `:uint64` | uint64 | 0 到 18446744073709551615 | `Params().GetUint64` |
| `:bool` | bool | "1" 或 "t" 或 "T" 或 "TRUE" 或 "true" 或 "True" 或 "0" 或 "f" 或 "F" 或 "FALSE" 或 "false" 或 "False" | `Params().GetBool` |
| `:alphabetical` | string | 小写或大写字母 | `Params().Get` |
| `:file` | string | 小写或大写字母, 数字, 下划线(_), 破折号(-), 点(.), 以及没有空格或其他对文件名无效的特殊字符 | `Params().Get` |
| `:path` | string | 任何东西, 都可以用斜杠(路径段)隔开, 但应该是路由路径的最后一部分 | `Params().Get` |

**用法:**

```go
app.Get("/users/{id:uint64}", func(ctx iris.Context){
    id := ctx.Params().GetUint64Default("id", 0)
    // [...]
})
```

| 内置函数 | 参数类型 |
| --- | --- |
| `regexp`(expr string) | :string |
| `prefix`(prefix string) | :string |
| `suffix`(suffix string) | :string |
| `contains`(s string) | :string |
| `min`(minValue int 或 int8 或 int16 或 int32 或 int64 或 uint8 或 uint16 或 uint32 或 uint64 或 float32 或 float64) | :string(char length), :int, :int8, :int16, :int32, :int64, :uint, :uint8, :uint16, :uint32, :uint64 |
| `max`(maxValue int 或 int8 或 int16 或 int32 或 int64 或 uint8 或 uint16 或 uint32 或 uint64 或 float32 或 float64) | :string(char length), :int, :int8, :int16, :int32, :int64, :uint, :uint8, :uint16, :uint32, :uint64 |
| `range`(minValue, maxValue int 或 int8 或 int16 或 int32 或 int64 或 uint8 或 uint16 或 uint32 或 uint64 或 float32 或 float64) | :int, :int8, :int16, :int32, :int64, :uint, :uint8, :uint16, :uint32, :uint64 |

**用法:**

```go
app.Get("/profile/{name:alphabetical max(255)}", func(ctx iris.Context){
    name := ctx.Params().Get("name")
    // len(name) <=255否则此路由将触发404 Not Found, 此处理程序将不会被执行
})
```

**独立完成:**

`RegisterFunc` 可以接受任何返回 `func(paramValue string) bool` 值或者只是一个 `func(string) bool` 值的函数。如果验证失败, 它将触发 `404` 或 `else` 关键字拥有的任何状态代码

```go
latLonExpr := "^-?[0-9]{1,3}(?:\\.[0-9]{1,10})?$"
latLonRegex, _ := regexp.Compile(latLonExpr)

// 将自定义无参数宏函数注册为 :string 参数类型。
// MatchString是一种 func(string) bool 类型, 因此我们按原样使用它
app.Macros().Get("string").RegisterFunc("coordinate", latLonRegex.MatchString)

app.Get("/coordinates/{lat:string coordinate()}/{lon:string coordinate()}",
func(ctx iris.Context) {
    ctx.Writef("Lat: %s | Lon: %s", ctx.Params().Get("lat"), ctx.Params().Get("lon"))
})
```

注册你的自定义宏函数, 它接受两个int参数

```go
app.Macros().Get("string").RegisterFunc("range",
func(minLength, maxLength int) func(string) bool {
    return func(paramValue string) bool {
        return len(paramValue) >= minLength && len(paramValue) <= maxLength
    }
})

app.Get("/limitchar/{name:string range(1,200) else 400}", func(ctx iris.Context) {
    name := ctx.Params().Get("name")
    ctx.Writef(`Hello %s | the name should be between 1 and 200 characters length
    otherwise this handler will not be executed`, name)
})
```

注册你的自定义宏函数, 它接受一个字符串切片[…, …]

```go
app.Macros().Get("string").RegisterFunc("has",
func(validNames []string) func(string) bool {
    return func(paramValue string) bool {
        for _, validName := range validNames {
            if validName == paramValue {
                return true
            }
        }

        return false
    }
})

app.Get("/static_validation/{name:string has([kataras,maropoulos])}",
func(ctx iris.Context) {
    name := ctx.Params().Get("name")
    ctx.Writef(`Hello %s | the name should be "kataras" or "maropoulos"
    otherwise this handler will not be executed`, name)
})
```

**实例代码:**

```go
func main() {
    app := iris.Default()

    // 这个处理程序将匹配/user/john, 但不匹配/user/或/user
    app.Get("/user/{name}", func(ctx iris.Context) {
        name := ctx.Params().Get("name")
        ctx.Writef("Hello %s", name)
    })

    // 这个处理程序将匹配/users/42
    // 但是不会匹配/users/-1, 因为uint应该大于0
    // /users 或 /users/ 都不会匹配
    app.Get("/users/{id:uint64}", func(ctx iris.Context) {
        id := ctx.Params().GetUint64Default("id", 0)
        ctx.Writef("User with ID: %d", id)
    })

    // 然而, 这个会匹配/user/john/send和/user/john/everything/else/
    // 但是不能匹配/user/john
    app.Post("/user/{name:string}/{action:path}", func(ctx iris.Context) {
        name := ctx.Params().Get("name")
        action := ctx.Params().Get("action")
        message := name + " is " + action
        ctx.WriteString(message)
    })

    app.Listen(":8080")
}
```

> 当参数类型缺失时，它默认为 `string` 类型，因此 `{name:string}` 和 `{name}` 指代的是相同的东西。
