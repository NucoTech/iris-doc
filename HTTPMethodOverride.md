# HTTP Method Override

在开发和升级REST API时, 使用特定的自定义HTTP请求头(如X-HTTP方法覆盖)非常的方便, 在部署基于REST API的web服务时, 您可能会需要**服务端**和**客户端**侧的访问限制

有些防火墙会不支持PUT, DELETE或者PATCH请求

[Method Override](https://github.com/kataras/iris/tree/master/middleware/methodoverride)允许您在客户端不支持的地方使用HTTP请求方法, 如PUT, DELETE

## 服务端(Server)

```go
package main

import (
    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/middleware/methodoverride"
)

func main() {
    app := iris.New() 

    mo := methodoverride.New( 
        // 默认为nil. 
        // 
        methodoverride.SaveOriginalMethod("_originalMethod"), 
        // 默认值. 
        // 
        // methodoverride.Methods(http.MethodPost), 
        // methodoverride.Headers("X-HTTP-Method",
        //                        "X-HTTP-Method-Override",
        //                        "X-Method-Override"), 
        // methodoverride.FormField("_method"), 
        // methodoverride.Query("_method"), 
    ) 
    // 用`WrapRouter`注册. 
    app.WrapRouter(mo)

    app.Post("/path", func(ctx iris.Context) {
        ctx.WriteString("post response")
    })

    app.Delete("/path", func(ctx iris.Context) {
        ctx.WriteString("delete response")
    })

    // [...app.Run]
}
```

## 客户端(Client)

```go
fetch("/path", {
    method: 'POST',
    headers: {
      "X-HTTP-Method": "DELETE"
    },
  })
  .then((resp)=>{
      // 响应体(response body)会被置为"delete response". 
 })).catch((err)=> { console.error(err) })
```
