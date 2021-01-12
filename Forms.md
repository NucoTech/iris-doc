# 表单(Forms)

表单、发送的数据和上传的文件可以通过以下的方法检索

```go
// FormValueDefault通过它的名称返回一个已解析的表单值
// 包含URL字段的查询参数和POST或PUT表单数据
// 如果没找到, 返回 "def"
FormValueDefault(name string, def string) string
// FormValue通过它的名称返回一个已解析的表单值
// 包含URL字段的查询参数和POST或PUT表单数据
FormValue(name string) string
// FormValues通过它的名称返回一个已解析的表单值
// 包含URL字段的查询参数和POST或PUT表单数据

// 默认表单的内存最大是32MB, 它可以由主配置器(main configuration)上的 `iris.WithPostMaxMemory` 配置器修改, 即改变 `app.Run` 的第二个参数

// 注意: 必须检查nil
FormValues() map[string][]string
// PostValueDefault返回从POST、PATCH或PUT通过名称解析的表单数据
//如果没找到, 返回 "def"
PostValueDefault(name string, def string) string
// PostValueDefault返回从POST、PATCH或PUT通过名称解析的表单数据
PostValue(name string) string
// PostValueTrim返回POST、PATCH或PUT通过名称解析的表单数据, 删去了尾部的空格
PostValueTrim(name string) string
// PostValueInt返回POST、PATCH或PUT通过名称解析的表单数据, 以int形式
// 如果没有找到, 返回-1和一个 "no-nil" 错误
PostValueInt(name string) (int, error)
// PostValueIntDefault返回POST、PATCH或PUT通过名称解析的表单数据, 以int形式
// 如果没有找到, 返回 "def"
PostValueIntDefault(name string, def int) int
// PostValueInt64返回POST、PATCH或PUT通过名称解析的表单数据, 以int64形式
// 如果没有找到, 返回-1和一个 "no-nil" 错误
PostValueInt64(name string) (int64, error)
// PostValueInt64Default返回POST、PATCH或PUT通过名称解析的表单数据, 以int64形式
// 如果没有找到, 返回 "def"
PostValueInt64Default(name string, def int64) int64
// PostValueInt64Default返回POST、PATCH或PUT通过名称解析的表单数据, 以float64形式
// 如果没有找到, 返回-1和一个 "no-nil" 错误
PostValueFloat64(name string) (float64, error)
// PostValueFloat64Default返回POST、PATCH或PUT通过名称解析的表单数据, 以float64形式
// 如果没有找到, 返回 "def"
PostValueFloat64Default(name string, def float64) float64
// PostValueBool返回POST、PATCH或PUT通过名称解析的表单数据, 以bool形式
// 如果没有找到或值为false, 返回false, 否则返回true
PostValueBool(name string) (bool, error)
// PostValues返回POST、PATCH或PUT通过名称解析的表单数据, 以string切片形式

// 默认表单的内存最大是32MB, 它可以由主配置器(main configuration)上的 `iris.WithPostMaxMemory` 配置器修改, 即改变 `app.Run` 的第二个参数
PostValues(name string) []string
// FormFile返回从客户端收到的第一个下载文件

// 默认表单的内存最大是32MB, 它可以由主配置器(main configuration)上的 `iris.WithPostMaxMemory` 配置器修改, 即改变 `app.Run` 的第二个参数
FormFile(key string) (multipart.File, *multipart.FileHeader, error)
```

## 多部分表单(Multipart/Urlencoded Form)

```go
func main() {
    app := iris.Default()

    app.Post("/form_post", func(ctx iris.Context) {
        message := ctx.FormValue("message")
        nick := ctx.FormValueDefault("nick", "anonymous")

        ctx.JSON(iris.Map{
            "status":  "posted",
            "message": message,
            "nick":    nick,
        })
    })

    app.Listen(":8080")
}
```

## 另一个例子: 查询 + 发送form

```http
POST /post?id=1234&page=1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=manu&message=this_is_great
```

```go
func main() {
    app := iris.Default()

    app.Post("/post", func(ctx iris.Context) {
        id := ctx.URLParam("id")
        page := ctx.URLParamDefault("page", "0")
        name := ctx.FormValue("name")
        message := ctx.FormValue("message")
        // 或者是POST, PUT & PATCH-only的 `ctx.PostValue` HTTP 方法.

        app.Logger().Infof("id: %s; page: %s; name: %s; message: %s",
            id, page, name, message)
    })

    app.Listen(":8080")
}
```

```http
id: 1234; page: 1; name: manu; message: this_is_great
```

## 上传文件

Iris Context为上传文件(将文件从请求文件数据保存到主机系统的硬盘上)提供了一个帮手, 下面有更多关于 `Context.UploadFormFiles` 的方法

UploadFormFiles上传任何从客户端接收到的文件到系统物理地址 "destDirectory"

第二个可选参数 "before" 让调用者在保存到磁盘之前可修改 `*mitipart.FileHeader` , 可以根据当前请求更改文件的名称, 文件头的所有选项都可以更改, 如果不需要, 可以忽略

注意，它不会检查请求体是否是流

如果由于操作系统的权限或 `net/http.ErrMissingFile` 方法没有接收到文件而导致至少一个新文件无法创建, 则返回一个长度为int64且非nil的错误

如果您想要接受文件并手动管理它们, 您可以使用 `Context.FormFile` , 并创建一个合适的复制函数，下面有通用的用法

默认表单的内存最大是32MB, 它可以由主配置器(main configuration)上的 `iris.WithPostMaxMemory` 配置器修改, 即改变 `app.Run` 的第二个参数

```go
UploadFormFiles(destDirectory string,
                before ...func(Context, *multipart.FileHeader)) (n int64, err error)
```

代码示例:

```go
const maxSize = 5 << 20 // 5MB

func main() {
    app := iris.Default()
    app.Post("/upload", iris.LimitRequestBodySize(maxSize), func(ctx iris.Context) {
        // UploadFormFiles
        // 上传任意数量的传入文件  (表单输入的"multiple").

        // 第二个可选参数, 可根据请求修改文件名
        // 在这个例子中, 我们将展示如何使用
        // 在上传文件加上用户的ip
        ctx.UploadFormFiles("./uploads", beforeSave)
    })

    app.Listen(":8080")
}

func beforeSave(ctx iris.Context, file *multipart.FileHeader) {
    ip := ctx.RemoteAddr()
    // 确保用一种可以用于文件名的方式格式化ip(简单情况)
    ip = strings.Replace(ip, ".", "_", -1)
    ip = strings.Replace(ip, ":", "_", -1)
    // 你可以用time.Now方法, 基于当前时间为文件加前缀或者后缀, 练习一下
    // 小贴士: unixTime := time.Now().Unix()
    //把 $ip- 作为文件名的前缀
    // 无需更多操作, 内部上传将会利用这个名称把文件保存到 "./uploads" 文件夹
    file.Filename = ip + "-" + file.Filename
}
```

怎样 `curl`:

```shell
curl -X POST http://localhost:8080/upload \
  -F "files[]=@./myfile.zip" \
  -F "files[]=@./mysecondfile.zip" \
  -H "Content-Type: multipart/form-data"
```

[戳我](https://github.com/kataras/iris/tree/master/_examples/request-body)获得更多的例子
[或者我](https://github.com/kataras/iris/tree/master/_examples/file-server)
