# 自动公共地址

## 自动带有TLS的公共地址

难道不想测试你的web应用程序在一个"现实世界环境"吗? 就跟公共环境、远端环境、取代localhost地址一样的那种!

有大量的第三方工具提供了这样的特性, 但是吧在我看来, [ngrok](https://github.com/inconshreveable/ngrok)在他们中是最好的。它非常流行并且经过了数年的测试, 就跟iris一样

Iris提供了ngrok集成。这个特性非常普遍，至今都很有用。它真正帮你快速在远程会议上给你的同事或者leader展示开发进度。

依据下面的步骤, 暂时的, 将你的本地iris web服务变成一个公共环境的

1. [下载ngrok](https://ngrok.io/), 将它加入你的环境变量
2. 简单地在你的`app.Run`传一个`WithTunneling`配置器
3. 完事, [冲!](https://www.facebook.com/iris.framework/photos/a.2420499271295384/3261189020559734/?type=3&theater)

![title](https://user-images.githubusercontent.com/22900943/81442996-42731800-917d-11ea-90da-7d6475a6b365.png)

- `ctx.Application().ConfigurationReadOnly().GetVHost()`返回一个公共域名值。没啥用, 但是是给你用的。大多数情况下你使用相对路径以取代绝对路径(或许你应该这么做)
- ngrok无所谓之前有没有启动, Iris框架智能地使用ngrok的[web API](https://ngrok.com/docs)去创建一个隧道

完整的`隧道(Tunneling)`配置项如下

```go
app.Listen(":8080", iris.WithConfiguration(
    iris.Configuration{
        Tunneling: iris.TunnelingConfiguration{
            AuthToken:    "my-ngrok-auth-client-token",
            Bin:          "/bin/path/for/ngrok",
            Region:       "eu",
            WebInterface: "127.0.0.1:4040",
            Tunnels: []iris.Tunnel{
                {
                    Name: "MyApp",
                    Addr: ":8080",
                },
            },
        }
    }
))
```

了解更多关于[配置项](../Configuration.md)
