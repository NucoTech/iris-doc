# HTTP引用(HTTP referrer)

Referrer-Policy HTTP消息头控制请求中应该包含多少referrer信息(通过referrer消息头发送)

在[developer.mozilla.org](developer.mozilla.org)中阅读更多

Iris使用[Shopify's goreferrer](https://github.com/Shopify/goreferrer/pull/27)包来显示 `Context.GetReferrer()` 方法

`GetReferrer` 方法从[链接](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)的 `Referer` (或 `Referrer`)消息头和url query参数中提取并返回信息

```go
GetReferrer() Referrer
```

当 `referrer` 形如此:

```go
type (
    Referrer struct {
        Type       ReferrerType
        Label      string
        URL        string
        Subdomain  string
        Domain     string
        Tld        string         
        Path       string              
        Query      string                 
        GoogleType ReferrerGoogleSearchType
    }
```

`ReferrerType` 是Referrer的枚举, 类型值(直接, 间接, 电子邮件, 搜索, 社交媒体), 可获得的类型为:

```go
ReferrerInvalid
ReferrerIndirect
ReferrerDirect
ReferrerEmail
ReferrerSearch
ReferrerSocial
```

`GoogleType` 可以是以下的一个:

```go
ReferrerNotGoogleSearch
ReferrerGoogleOrganicSearch
ReferrerGoogleAdwords
```

## 例子(Example)

```go
package main

import "github.com/kataras/iris/v12"

func main() {
    app := iris.New()

    app.Get("/", func(ctx iris.Context) {
        r := ctx.GetReferrer()
        switch r.Type {
        case iris.ReferrerSearch:
            ctx.Writef("Search %s: %s\n", r.Label, r.Query)
            ctx.Writef("Google: %s\n", r.GoogleType)
        case iris.ReferrerSocial:
            ctx.Writef("Social %s\n", r.Label)
        case iris.ReferrerIndirect:
            ctx.Writef("Indirect: %s\n", r.URL)
        }
    })

    app.Listen(":8080")
}
```

怎样 `crul`:

```shell
curl http://localhost:8080?\
referrer=https://twitter.com/Xinterio/status/1023566830974251008

curl http://localhost:8080?\
referrer=https://www.google.com/search?q=Top+6+golang+web+frameworks\
&oq=Top+6+golang+web+frameworks
```
