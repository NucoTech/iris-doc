# 国际化

## 介绍

国际化特性提供了一个便捷的方法同在不同的语言里检索字符串, 允许你在应用程序中很简单的支持大量的语言。语言字符串被储存在`./locales`目录下的文件中。在这个目录下应该为应用程序所支持的每一种语言建立一个子目录

```text
│   main.go
└───locales
    ├───el-GR
    │       home.yml
    ├───en-US
    │       home.yml
    └───zh-CN
            home.yml
```

你应用程序的默认语言是第一个被注册的语言

```go
app := iris.New()

// 第一个参数: Glob 文件路径模式,
// 第二个可变参数: 自定义语言标签,
// 第一中被设置为 默认/失败返回(fallback)
app.I18n.Load("./locales/*/*", "en-US", "el-GR", "zh-CN")
```

或者如果你通过下面来加载所有的语言

```go
app.I18n.Load("./locales/*/*")
```

然后通过下面这来设置默认语言

```go
app.I18n.SetDefault("en-US")
```

## 加载嵌入式本土化

你可能想在你应用程序运行中通过go-bindata工具使用嵌入式本土化

1. 下载go-bindata工具, 例如 `go get -u github.com/go-bindata/go-bindata/...`
2. 将嵌入式本土化文件加入到你的应用程序 `go-bindata -o locales.go ./locales/...`
3. 使用`LoadAssets`方法初始化并且加载语言 ^ `AssetNames`和`Asset`函数式由`gin-bindata`生成的

```go
app.I18n.LoadAssets(AssetNames, Asset, "en-US", "el-GR", "zh-CN")
```

## 定义翻译

每一个文件应当包含翻译过的文本或者模板值的键

### Fmt风格

```yml
hi: "Hi %s"
```

### 模板风格

```yml
hi: "Hi {{ .Name }}"
# 模板函数也被支持
# hi: "Hi {{sayHi .Name}}
```

### INI Sections

```ini
[cart]
checkout = ολοκλήρωση παραγγελίας - {{.Param}}
```

> YAML, TOML, JSON, INI文件

## 确定当前的本土化

你可能使用`context.GetLocale`方法去确定当前的本土化, 或者检查是否本土化已经被赋值

```go
func(ctx iris.Context) {
    locale := ctx.GetLocale()
    // [...]
}
```

**本土化**接口长这样

```go
// 本土化是一个接口返回 `Localizer.GetLocale` 方法
// 它基于 "键" 或者格式化提供翻译服务, 参见 `GetMessage`
type Locale interface {
    // Index返回当前本土化在语言列表中的索引
    Index() int
    // Tag返回附加在这个本土化上完整的语言标签
    // 在不同的本土化之间应当是独一无二的
    Tag() *language.Tag
    // Language 应该当返回这个 `本土化(Locale)` 确切的语言代码
    // 用户可以在 `New` 函数中提供
    //
    // 与 `Tag().String()` 相同但是它是静态的
    Language() string
    // GetMessage 应当返回基于给定"键"的被翻译的文本
    GetMessage(key string, args ...interface{}) string
}
```

## 检索翻译

在这个请求中, 使用`context.Tr`方法作为获取翻译文本的简便方法

```go
func(ctx iris.Context) {
    text := ctx.Tr("cart.checkout", iris.Map{"Param": "a value"})
    // [...]
}
```

## 在视图层(Views)中

```go
func(ctx iris.Context) {
    ctx.View("index.html", iris.Map{
        "tr": ctx.Tr,
    })
}
```

```html
{{ call .tr "hi" "John Doe" }}
```

## [示例](https://github.com/kataras/iris/tree/master/_examples/i18n)

```go
package main

import (
    "github.com/kataras/iris/v12"
)

func newApp() *iris.Application {
    app := iris.New()

    // 设置国际化
    // 第一个参数: Glob 路径模式,
    // 第二个可变参数: 自定义语言标签, 第一个为 默认/失败返回
    app.I18n.Load("./locales/*/*.ini", "en-US", "el-GR", "zh-CN")
    // app.I18n.LoadAssets for go-bindata.

    // 默认值:
    // app.I18n.URLParameter = "lang"
    // app.I18n.Subdomain = true
    //
    // 设为false不允许失败重定向路径(本地),
    // 参考 https://github.com/kataras/iris/issues/1369.
    // app.I18n.PathRedirect = true

    app.Get("/", func(ctx iris.Context) {
        hi := ctx.Tr("hi", "iris")

        locale := ctx.GetLocale()

        ctx.Writef("From the language %s translated output: %s", locale.Language(), hi)
    })

    app.Get("/some-path", func(ctx iris.Context) {
        ctx.Writef("%s", ctx.Tr("hi", "iris"))
    })

    app.Get("/other", func(ctx iris.Context) {
        language := ctx.GetLocale().Language()

        fromFirstFileValue := ctx.Tr("key1")
        fromSecondFileValue := ctx.Tr("key2")
        ctx.Writef("From the language: %s, translated output:\n%s=%s\n%s=%s",
            language, "key1", fromFirstFileValue,
            "key2", fromSecondFileValue)
    })

    // 在你的视图层使用
    view := iris.HTML("./views", ".html")
    app.RegisterView(view)

    app.Get("/templates", func(ctx iris.Context) {
        ctx.View("index.html", iris.Map{
            "tr": ctx.Tr, // 文字, 变量... {call .tr "hi" "iris"}}
        })

        // 注意,
        // Iris会默认添加一个"tr"全局模板函数,
        // 这种方式唯一的区别在于你在你的模板中调用
        // 它接受一个语言代码作为它的第一个参数: {{ tr "el-GR" "hi" "iris"}}
    })

    return app
}

func main() {
    app := newApp()

    // go to http://localhost:8080/el-gr/some-path
    // ^ (通过前缀)
    //
    // or http://el.mydomain.com:8080/some-path
    // ^ (通过子域名 - 修改hosts文件在本地测试)
    //
    // or http://localhost:8080/zh-CN/templates
    // ^ (通过路径前缀的首字母大写)
    //
    // or http://localhost:8080/some-path?lang=el-GR
    // ^ (通过url参数)
    //
    // or http://localhost:8080 (默认为 en-US)
    // or http://localhost:8080/?lang=zh-CN
    //
    // go to http://localhost:8080/other?lang=el-GR
    // or http://localhost:8080/other (默认为 en-US)
    // or http://localhost:8080/other?lang=en-US
    //
    // or 使用cookies设置语言
    app.Listen(":8080", iris.WithSitemap("http://localhost:8080"))
}
```

## Sitemap

[Sitemap](https://github.com/kataras/iris/wiki/Sitemap)翻译对每一个路由是自动设置的, 通过路径前缀实现如果`app.I18n.PathRedirect`是true, 通过子域名如果`app.I18n.Subdomain`是true, 通过URL query参数如果`app.I18n.URLParameter`不为空

了解更多 [https://support.google.com/webmasters/answer/189077?hl=en](https://support.google.com/webmasters/answer/189077?hl=en)

```shell
GET http://localhost:8080/sitemap.xml
```

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">
    <url>
        <loc>http://localhost:8080/</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="http://localhost:8080/"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="el-GR" href="http://localhost:8080/el-GR/"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="zh-CN" href="http://localhost:8080/zh-CN/"></xhtml:link>
    </url>
    <url>
        <loc>http://localhost:8080/some-path</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="http://localhost:8080/some-path"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="el-GR" href="http://localhost:8080/el-GR/some-path"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="zh-CN" href="http://localhost:8080/zh-CN/some-path"></xhtml:link>
    </url>
    <url>
        <loc>http://localhost:8080/other</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="http://localhost:8080/other"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="el-GR" href="http://localhost:8080/el-GR/other"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="zh-CN" href="http://localhost:8080/zh-CN/other"></xhtml:link>
    </url>
    <url>
        <loc>http://localhost:8080/templates</loc>
        <xhtml:link rel="alternate" hreflang="en-US" href="http://localhost:8080/templates"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="el-GR" href="http://localhost:8080/el-GR/templates"></xhtml:link>
        <xhtml:link rel="alternate" hreflang="zh-CN" href="http://localhost:8080/zh-CN/templates"></xhtml:link>
    </url>
</urlset>
```
