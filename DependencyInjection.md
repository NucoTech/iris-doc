# 依赖注入(dependency injection)

Iris通过请求处理程序和基于返回值的服务器应答, 为依赖注入提供一流的支持

## 依赖注入(Dependency Injection)

使用Iris, 您可以得到真正安全的绑定, 它贼快, 接近原生处理器的性能, 因为我们可以在服务器上线之前就预先分配必要信息

依赖项可以是函数, 也可以是静态值, 函数依赖项也可以接受先前注册的依赖项作为其输入参数

代码示例:

```go
func printFromTo(from, to string) string { return "message" }

// [...]
app.ConfigureContainer(func (api *iris.APIContainer){
    api.Get("/{from}/{to}", printFromTo)
})
```

正如你看到的, `Iris.Context` 输入是完全可选的, Iris足够聪明去绑定, 没有任何问题

## 概述(OverView)

从路由到处理器最常见的场景是:

- 接收一个或多个路径参数并请求数据, 即一个payload
- 发送一个响应, 即一个payload(JSON, XML...)
  
新型Iris依赖注入特性比上一版快了33.2%, 这将进一步降低处理程序和具有依赖关系的处理程序之间的性能成本, 让我们在使用新的 `组.ConfigureContainer(builder ...func(*iris.APIContainer)) *APIContainer` 方法(该方法返回 `Handle(method, relativePath string, handlersFn ...interface{}) *Route` 和 `RegisterDependency` 一类的方法)时获得更安全和更好性能的体验

看看在使用Iris时, 您的代码库是多么地整洁:

```go
package main

import "github.com/kataras/iris/v12"

type (
    testInput struct {
        Email string `json:"email"`
    }

    testOutput struct {
        ID   int    `json:"id"`
        Name string `json:"name"`
    }
)

func handler(id int, in testInput) testOutput {
    return testOutput{
        ID:   id,
        Name: in.Email,
    }
}

func main() {
    app := iris.New()
    app.ConfigureContainer(func(api *iris.APIContainer) {
        api.Post("/{id:int}", handler)
    })
    app.Listen(":5000", iris.WithOptimizations)
}
```

眼见为实, 您看, 没有 `ctx.ReadJSON(&v)` 和 `ctx.JSON(send)` 甚至是 `error` 需要处理, 这是一个巨大的解脱, 但如果您需要, 您仍然可以控制这些, 甚至是依赖项产生的错误, 以下是新型 `组.ConfigureContainer()` 方法和所需参数的列表

```go
// 容器(Container)控制该组具有依赖特性的DI容器
// 使用它手动转换函数或结构体(控制器)到处理程序
Container *hero.Container
```

```go
// OnError为该组的DI hero容器和它的处理程序(或称控制器)添加一个错误处理程序
// 在该组的hero 处理程序或处理程序本身依赖注入的过程中
// "errorHandler" 处理一切可能发生的错误并且返回
OnError(errorHandler func(iris.Context, error))
```

```go
// RegisterDependency添加一个依赖项, 可以是一个单独的结构体值或者功能
// 遵从以下语法:
// * <T> {structValue}
// * func(accepts <T>)                                 returns <D> or (<D>, error)
// * func(accepts iris.Context)                        returns <D> or (<D>, error)
//
// 依赖项可以接收以前注册的依赖项, 并返回新的依赖项或已经更新的依赖项
// * func(accepts1 <D>, accepts2 <T>)                  returns <E> or (<E>, error) or error
// * func(acceptsPathParameter1 string, id uint64)     returns <T> or (<T>, error)
//
// 方法:
//
// - RegisterDependency(loggerService{prefix: "dev"})
// - RegisterDependency(func(ctx iris.Context) User {...})
// - RegisterDependency(func(User) OtherResponse {...})
RegisterDependency(dependency interface{})

// UseResultHandler向容器添加一个结果处理程序
// 结果处理程序可用来注入返回的结构体值, 或替换默认的渲染器
UseResultHandler(handler func(next iris.ResultHandler) iris.ResultHandler)
```

<details>
  <summary>ResultHandler</summary>
  <pre><code> 
     type ResultHandler func(ctx iris.Context, v interface{}) error
  </code></pre>
</details>

```go
// Use和通用组相同的 "Use" 但是它接收动态功能作为它的 "handlersFn" 输入
Use(handlersFn ...interface{})
// Done和通用组相同的 "Done" 但是它接收动态功能作为它的 "handlersFn" 输入
Done(handlersFn ...interface{})
```

```go
// Handle与通用的组的 `Handle` 相同, 但它接受一个或多个"handlersFn"函数, 
// 每个函数都可以接收任何与该组注册容器的 ` Dependencies` 和任何输出结果相匹配的输入结果
// 就像自定义structs <T>,  string,  []byte, int, error, 
// 上面的一个组合, hero.Result(hero.view | hero.Response)等等

//hero处理程序通常不需要接收 `Context` , 因此, `handlersFn` 将在未手动调用时自动调用 `ctx.Next()`

//如果要停止执行, 而不是执行下一个 "handlersFn" , end-developer 需要输出一个错误并且返回 `iris.ErrStopExecution`
Handle(method, relativePath string, handlersFn ...interface{}) *Route

// Get注册一个GET路由, 与 `Handle("GET", relativePath, handlersFn....)` 一样.
Get(relativePath string, handlersFn ...interface{}) *Route
```

下面是可以直接作为输入参数使用的内置依赖项列表:

| 类型 | 映射到 |
| --- | --- |
| [*mvc.Application](https://pkg.go.dev/github.com/kataras/iris/v12/mvc?tab=doc#Application) | Current MVC Application |
| [iris.Context](https://pkg.go.dev/github.com/kataras/iris/v12/context?tab=doc#Context) | Current Iris Context |
| [*sessions.Session](https://pkg.go.dev/github.com/kataras/iris/v12/sessions?tab=doc#Session) | Current Iris Session |
| [context.Context](https://golang.org/pkg/context/#Context) | [ctx.Request().Context()](https://golang.org/pkg/net/http/#Request.Context) |
| [*http.Request](https://golang.org/pkg/net/http/#Request) | ctx.Request() |
| [http.ResponseWriter](https://golang.org/pkg/net/http/#ResponseWriter) | ctx.ResponseWriter() |
| [http.Header](https://golang.org/pkg/net/http/#Header) | ctx.Request().Header |
| [time.Time](https://golang.org/pkg/time/#Time) | time.Now() |
| string, | |
| int, int8, int16, int32, int64,| |
| uint, uint8, uint16, uint32, uint64,| |
| float, float32, float64,| |
| bool, | |
| slice | [Path Parameter](https://github.com/kataras/iris/wiki/Routing-path-parameter-types) |
| Struct | [Request Body](https://github.com/kataras/iris/tree/master/_examples/request-body) of `JSON` , `XML` ,  `YAML` , `Form` , `URL Query` , `Protobuf` , `MsgPack` |

## 请求、响应和路径参数

1.为客户端请求体和服务器响应声明Go的类型

```go
type (
    request struct {
        Firstname string `json:"firstname"`
        Lastname  string `json:"lastname"`
    }

    response struct {
        ID      uint64 `json:"id"`
        Message string `json:"message"`
    }
)
```

2.创建路由处理程序

路径参数和请求体会自动绑定

- **id uint64** 与 "id:uint64" 绑定
- **输入请求** 与客户端请求数据(比如JSON)绑定

```go
func updateUser(id uint64, input request) response {
    return response{
        ID:      id,
        Message: "User updated successfully",
    }
}
```

3.为每个组配置容器并注册路由

```go
app.组("/user").ConfigureContainer(container)

func container(api *iris.APIContainer) {
    api.Put("/{id:uint64}", updateUser)
}
```

4.模拟向服务器发送数据并显示响应的[客户端](https://curl.haxx.se/download.html)请求

```shell
curl --request PUT -d '{"firstanme":"John","lastname":"Doe"}' http://localhost:8080/user/42
```

```go
{
    "id": 42,
    "message": "User updated successfully"
}
```

## 自定义预检(Custom Preflight)

在我们继续下一节注册依赖项之前, 您可能想了解如何在响应发送到客户端之前通过 `iris.Context` 定制它

在将响应发送给客户端之前, 服务端将自动执行函数的输出结构体值的 `Preflight(iris.Context) error` 方法。

例如, 您希望根据处理程序内部的自定义逻辑触发不同的HTTP状态码, 并在发送到客户端之前修改值(响应体)本身。您的响应类型应该包含如下所示的 `Preflight` 方法

```go
type response struct {
    ID      uint64 `json:"id,omitempty"`
    Message string `json:"message"`
    Code    int    `json:"code"`
    Timestamp int64 `json:"timestamp,omitempty"`
}

func (r *response) Preflight(ctx iris.Context) error {
    if r.ID > 0 {
        r.Timestamp = time.Now().Unix()
    }

    ctx.StatusCode(r.Code)
    return nil
}
```

现在, 每一个处理程序都会自动返回一个将会调用 `response.Preflight` 方法的 `*response` 值

```go
func deleteUser(db *sql.DB, id uint64) *response {
    // [...自定义逻辑]

    return &response{
        Message: "User has been marked for deletion",
        Code: iris.StatusAccepted,
    }
}
```

如果您注册了路由并发出请求, 您应看到如下输出: 时间戳已填充, 客户端将接收到的响应的HTTP状态码为202(状态已接受)

```go
{
  "message": "User has been marked for deletion",
  "code": 202,
  "timestamp": 1583313026
}
```

## 注册依赖项(Register Dependencies)

1.导入包与数据库交互, go-sqlite3包是一个[SQLite](https://www.sqlite.org/index.html)的数据库控制器

```go
import "database/sql"
import _ "github.com/mattn/go-sqlite3"
```

2.配置容器([见上方](#请求响应和路径参数)), 注册依赖性, 处理程序需要一个 *sql.DB实例

```go
localDB, _ := sql.Open("sqlite3", "./foo.db")
api.RegisterDependency(localDB)
```

3.注册路由来创建用户

```go
api.Post("/{id:uint64}", createUser)
```

4.创建用户处理程序

处理程序接受数据库和一些客户端请求数据, 如JSON、Protobuf、表单、URL查询等, 然后返回一个响应

```go
func createUser(db *sql.DB, user request) *response {
    // [自定义数据库的交互逻辑]
    userID, err := db.CreateUser(user)
    if err != nil {
        return &response{
            Message: err.Error(),
            Code: iris.StatusInternalServerError,
        }
    }

    return &response{
        ID:      userID,
        Message: "User created",
        Code:    iris.StatusCreated,
    }
}
```

5.模拟一个[客户端](https://curl.haxx.se/download.html)来创建用户

```shell
# JSON
curl --request POST -d '{"firstname":"John","lastname":"Doe"}' \
--header 'Content-Type: application/json' \
http://localhost:8080/user
```

```shell
# Form (multipart)
curl --request POST 'http://localhost:8080/users' \
--header 'Content-Type: multipart/form-data' \
--form 'firstname=John' \
--form 'lastname=Doe'
```

```shell
# Form (URL-encoded)
curl --request POST 'http://localhost:8080/users' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'firstname=John' \
--data-urlencode 'lastname=Doe'
```

```shell
# URL Query
curl --request POST 'http://localhost:8080/users?firstname=John&lastname=Doe'
```

响应:

```go
{
    "id": 42,
    "message": "User created",
    "code": 201,
    "timestamp": 1583313026
}
```

## 返回值

- 如果返回值是string, 那么它将把该字符串作为响应体发送

- 如果它是一个int, 那么它会把它作为一个状态码发送

- 如果它是一个错误, 那么它将设置一个坏请求, 并将该错误作为其原因

- 如果它是一个错误和一个整数, 那么错误代码是该整数, 而不是400(坏请求)

- 如果它是一个自定义结构, 那么当内容类型头还没有设置时, 它将作为JSON发送

- 如果它是一个自定义结构和一个字符串, 那么第二个输出值是string, 它将是Content-Type, 以此类推

| 类型 | 回复 |
| --- | --- |
| string | body |
| string, string | content-type, body |
| string, int | body, status code |
| int | status code |
| int, string | status code, body |
| error | if not nil, bad request |
| any, bool | if false then fires not found |
| <Τ> | JSON body |
| <Τ>, string | body, content-type |
| <Τ>, error | JSON body or bad request |
| <Τ>, int | JSON body, status code |
| [Result](https://godoc.org/github.com/kataras/iris/hero#Result) | calls its Dispatch method |
| [PreflightResult](https://godoc.org/github.com/kataras/iris/hero#PreflightResult) | calls its Preflight method |

`<T>` 代表任意结构体的值

```go
type response struct {
    ID      uint64 `json:"id,omitempty"`
    Message string `json:"message"`
    Code    int    `json:"code"`
    Timestamp int64 `json:"timestamp,omitempty"`
}

func (r *response) Preflight(ctx iris.Context) error {
    if r.ID > 0 {
        r.Timestamp = time.Now().Unix()
    }

    ctx.StatusCode(r.Code)
    return nil
}

func deleteUser(db *sql.DB, id uint64) *response {
    // [...自定义逻辑]

    return &response{
        Message: "User has been marked for deletion",
        Code: iris.StatusAccepted,
    }
}
```

稍后, 您将看到这些知识怎样帮助您使用MVC架构模式(Iris为它提供了超棒的API)构建应用
