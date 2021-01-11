# Content Negotiation

**有时, 服务器应用程序需要在同一个URI上为资源的不同表示提供服务**, 当然, 这可以手动完成, 手动检查 `Accept` 请求头并推送内容的请求形式, 然而, 当你的app控制更多的资源和不同类型的表示时, 这相当苦逼, 因为你可能需要检查 `Accept-Charset` , `Accept-Encoding` , 设置一些服务端优先级, 处理程序正确处理错误等等

在Go中有一些web框架已经在努力地实现这样的功能，但他们做得不对:

- 根本不处理 Accept-Charset
- 根本不处理 Accept-Encoding
- 不会像RFC提议的那样发送错误状态码(406不可接受)等等

但是万幸, **iris始终坚持最佳的实践效果和web标准**

基于以下:

- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation)
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept)
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Charset](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Charset)
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding)

通过以下实现:

- [https://github.com/kataras/iris/pull/1316/commits/8ee0de51c593fe0483fbea38117c3c88e065f2ef#diff-15cce7299aae8810bcab9b0bf9a2fdb1](https://github.com/kataras/iris/pull/1316/commits/8ee0de51c593fe0483fbea38117c3c88e065f2ef#diff-15cce7299aae8810bcab9b0bf9a2fdb1)

## 例子(Example)

```go
type testdata struct {
    Name string `json:"name" xml:"Name"`
    Age  int    `json:"age" xml:"Age"`
}
```

用 "gzip" 编码算法将资源渲染为application/json 或者 text/xml 或者 application/xml

- 当客户端的Accept头包含他们其中之一时
- 或者JSON(第一个声明), 如果Accept为空时
- 并且当客户端Accept-Encoding头包含 "gzip" 或者为空时

```go
app.Get("/resource", func(ctx iris.Context) {
    data := testdata{
        Name: "test name",
        Age:  26,
    }

        ctx.Negotiation().JSON().XML().EncodingGzip()

    _, err := ctx.Negotiate(data)
    if err != nil {
        ctx.Writef("%v", err)
    }
})
```

**或者**在中间件中定义, 并在最终处理程序中协商为nil

```go
ctx.Negotiation().JSON(data).XML(data).Any("content for */*")
ctx.Negotiate(nil)
```

```go
app.Get("/resource2", func(ctx iris.Context) {
    jsonAndXML := testdata{
        Name: "test name",
        Age:  26,
    }

    ctx.Negotiation().
        JSON(jsonAndXML).
        XML(jsonAndXML).
        HTML("<h1>Test Name</h1><h2>Age 26</h2>")

    ctx.Negotiate(nil)
})
```

[点我阅读所有例子](https://github.com/kataras/iris/blob/master/_examples/response-writer/content-negotiation/main.go#L22)

## 文档

[Context.Negotiation](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3342)方法会创建一次并且返回协商构建器, 来构建服务端特定的内容类型, 字符集和编码算法的可用优先级内容

`Context.Negotiation() *context.NegotiationBuilder`

[Context.Negotiate](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3402)方法用于在同一个URI上为资源的不同表示提供服务, 当未匹配到mime类型时,返回 `context.ErrContentNotSupported`

- "v" 可以是单个的[iris.N](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3298-L3309)结构体的值

- "v" 可以是[Context.ContentSelector](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3272)接口的任何值

- "v" 可以是[Context.ContentNegotiator](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3281)接口的任何值

- "v" 可以是struct(JSON, JSONP, XML, YAML)或string(文本，HTML)或[]字节(Markdown，二进制)或[]byte的任意值，任何匹配的mime类型。

- 如果 "v" 是nil, `context.negotiation()` 构建器的内容将被使用，否则 "v" 将覆盖构建器的内容(服务端mime类型仍然通过其注册的、支持的mime列表进行检索)

- 通过 [Negotiation().MIME.Text.JSON.XML.HTML....](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3500-L3621) 设置mime类型优先级

- 通过 [Negotiation().charset(…)](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3640) 设置字符优先级

- 通过 [Negotiation().encoding(…)](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3652-L3665) 设置编码算法优先级。

- 修改被接受的 [Negotiation().Accept./Override()/.XML().JSON().Charset(...).Encoding(...)....](https://github.com/kataras/iris/blob/8ee0de51c593fe0483fbea38117c3c88e065f2ef/context/context.go#L3774-L3877)

```go
Context.Negotiate(v interface{}) (int, error)
```
