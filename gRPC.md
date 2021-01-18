# Grpc

gRPC*是一个现代的开源高性能远程过程调用框架, 可以在任何环境下运行。它可以高效地连接数据中心内和跨数据中心的服务, 支持可插拔的负载均衡、跟踪、健康检查和身份验证。它也适用于分布式计算的最后一步, 将设备、移动应用程序和浏览器连接到后端服务

## gRPC

Iris和gRPC集成在 [mvc](https://github.com/kataras/iris/tree/master/mvc) 包中

你是否曾有过将你的应用或它的部分从HTTP转换到gGRPC时出现问题? 你是否曾希望你的gRPC服务也有像样的HTTP框架支持? 现在, 有了Iris, 你就拥有了两个领域中最好的框架。不需要修改您现有的gRPC服务代码, 您就可以通过控制器(您的服务结构体的值)将它们注册为Iris HTTP路由

![conversation](https://github.com/kataras/iris/wiki/_assets/grpc-compatible-question.png)

[戳我](https://github.com/kataras/iris/issues/1449#issuecomment-623260695)看我们的更多对话

## 一步一步来

我们将遵循[官方helloworld gRPC实例](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) ,如果您用gPRC服务, 那您可以跳过前五步

1.让我们来编写请求和响应的原型表

```go
syntax = "proto3";

package helloworld;

// 定义欢迎服务
service Greeter {
  // 发送一个欢迎
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 获得用户名称的请求信息
message HelloRequest {
  string name = 1;
}

// 包含欢迎的响应信息
message HelloReply {
  string message = 1;
}
```

2.安装原型的Go插件

```shell
go get -u github.com/golang/protobuf/protoc-gen-go
```

3.从上面的helloworld.proto文件生成go文件

```shell
protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld
```

4.像往常一样生成gRPC, 不管有没有Iris

```go
import (
    // [...]
    pb "myapp/helloworld"
    "context"
)
```

```go
type Greeter struct { }

// SayHello实现了helloworld.GreeterServer.
func (c *Greeter) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}
```

Iris自动绑定标准的 "context" `context.Context` 到 `iris.Context.Request().Context()` 和任何其他结构(没有映射到注册的依赖作为payload(取决于请求), 如XML, YAML, 查询, 表单, JSON, Protobuf)

5.在gRPC服务器上注册您的服务

```go
import (
    // [...]
    pb "myapp/helloworld"
    "google.golang.org/grpc"
)
```

```go
grpcServer := grpc.NewServer()

myService := &Greeter{}
pb.RegisterGreeterServer(grpcServer, myService)
```

6.注册 `myService` 到Iris

`mvc.New(party).Handle(ctrl, mvc.GRPC{...})` 选项允许每一组注册gRPC服务(不需要完整的包装器), 也可以严格的选择访问仅gRPC的客户端

为gRPC服务注册MVC应用程序控制器, 由于 `ServiceName` 不同, 你可以在同一组或应用程序中绑定尽可能多的mvc gRpc服务

```go
import (
    // [...]
    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/mvc"
)
```

```go
app := iris.New()

rootApp := mvc.New(app)
rootApp.Handle(myService, mvc.GRPC{
    Server:      grpcServer,           // Required.
    ServiceName: "helloworld.Greeter", // Required.
    Strict:      false,
})
```

7.生成TLS Keys

Iris服务**必须跑在TLS**(gRPC的一个要求)下

```shell
openssl genrsa -out server.key 2048
openssl req -new -x509 -sha256 -key server.key -out server.crt -days 3650
```

8.监听和服务

```go
app.Run(iris.TLS(":443", "server.crt", "server.key"))
```

POST: `https://localhost:443/helloworld.Greeter/SayHello` 发送数据 `{"name": "John"}`将会得到输出 `{"message": "Hello John"}`

HTTP客户端和gRPC客户端都可以与我们的Iris+gRPC服务交流

## 练习文件

完整的服务器, 客户端和测试代码可以[点我](https://github.com/kataras/iris/tree/master/_examples/mvc/grpc-compatible)找到
