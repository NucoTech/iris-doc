# 请求认证(Request authentication)

Iris通过它的[JWT中间件](https://github.com/iris-contrib/middleware/tree/master/jwt)提供身份验证, 在本节中, 您将学习如何在Iris中使用JWT的基础知识

1.通过执行以下shell命令安装:

```shell
$go get github.com/iris-contrib/middleware/jwt
```

2.要创建一个JWT中间件, 需要使用 `jwt.New` 功能, 本例通过 `token` 这个url参数提取token, 通过身份验证的客户端应该使用签名token设置, jwt中间件的默认行为是通过 `Authentication: Bearer $TOKEN` 消息头提取token值

jwt中间件有三种方法验证token

- `Serve` 方法, 这是一个 `iris.Handler`
- `CheckJWT(iris.Context)bool`
- 助手检索通过验证的token - `Get(iris.Context) *jwt.Token`

3.要注册中间件, 你只需要预先将jwt `j.Serve` 中间件添加至一组特定的路由, 单个路由或全局路由, 例如:`app.Get("/secured", j.Serve, myAuthenticatedHandler)`

4.要在一个处理程序中生成一个token, 该处理程序可接受用户负载并响应签名token, 稍后可以通过客户端请求头或者URL参数发送token, 例如: `jwt.NewToken` 或 `jwt.NewTokenWithClaims`

## 例子(Example)

```go
import (
    "github.com/kataras/iris/v12"
    "github.com/iris-contrib/middleware/jwt"
)

func getTokenHandler(ctx iris.Context) {
    token := jwt.NewTokenWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "foo": "bar",
    })

    // 使用secret将完整的编码token标记为string
    tokenString, _ := token.SignedString([]byte("My Secret"))

    ctx.HTML(`Token: ` + tokenString + `<br/><br/>
    <a href="/secured?token=` + tokenString + `">/secured?token=` + tokenString + `</a>`)
}

func myAuthenticatedHandler(ctx iris.Context) {
    user := ctx.Values().Get("jwt").(*jwt.Token)

    ctx.Writef("This is an authenticated request\n")
    ctx.Writef("Claim content:\n")

    foobar := user.Claims.(jwt.MapClaims)
    for key, value := range foobar {
        ctx.Writef("%s = %s", key, value)
    }
}

func main(){
    app := iris.New()

    j := jwt.New(jwt.Config{
        // 通过token提取URL参数
        Extractor: jwt.FromParameter("token"),

        ValidationKeyGetter: func(token *jwt.Token) (interface{}, error) {
            return []byte("My Secret"), nil
        },
        SigningMethod: jwt.SigningMethodHS256,
    })

    app.Get("/", getTokenHandler)
    app.Get("/secured", j.Serve, myAuthenticatedHandler)
    app.Listen(":8080")
}
```

你可以将任何负载作为"声明"
