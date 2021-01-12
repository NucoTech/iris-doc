# Model验证(Model validation)

Iris十分英明的没有内置数据的验证功能, 不过, 它确实允许你附加一个验证器, 它会自动调用 `Context.ReadJSON, ReadXML...` 在这个例子中, 我们将学习如何使用 [go-playground/validator/v10](https://github.com/go-playground/validator) 进行请求体的验证

来[这儿](https://pkg.go.dev/github.com/go-playground/validator/v10@v10.2.0?tab=doc)检查完整的文档和结构体标志用法

```shell
go get github.com/go-playground/validator/v10@latest
```

请注意, 您需要在想要绑定的所有字段上设置响应的绑定标记, 例如, 当从JSON绑定时, 设置 `json:"fieldname"`

```go
package main

import (
    "fmt"

    "github.com/kataras/iris/v12"

    "github.com/go-playground/validator/v10"
)

// 用户信息
type User struct {
    FirstName      string     `json:"fname"`
    LastName       string     `json:"lname"`
    Age            uint8      `json:"age" validate:"gte=0,lte=130"`
    Email          string     `json:"email" validate:"required,email"`
    FavouriteColor string     `json:"favColor" validate:"hexcolor|rgb|rgba"`
    Addresses      []*Address `json:"addresses" validate:"required,dive,required"`
}

// 用户地址信息
type Address struct {
    Street string `json:"street" validate:"required"`
    City   string `json:"city" validate:"required"`
    Planet string `json:"planet" validate:"required"`
    Phone  string `json:"phone" validate:"required"`
}

func main() {
    // 创建一个Validator实例, 它会缓存结构体信息.
    v := validator.New()

    // 为 'User' 注册一个自定义结构验证
    // 注意: 只需要注册一个非指针类型的 'User' 验证器
    // 在类型检查期间内部取消引用
    v.RegisterStructValidation(UserStructLevelValidation, User{})

    app := iris.New()
    // 将validator注册到Iris应用
    app.Validator = v

    app.Post("/user", func(ctx iris.Context) {
        var user User

        // 若验证输入错误, 返回InvalidValidationError
        // nil or ValidationErrors ( []FieldError )
        err := vctx.ReadJSON(&user)
        if err != nil {
            // 只有您的代码可生成的时候才需要此检查 
            //一个无效的验证值,例如interface with nil
            //大多数人包括我自己通常不会有这样的代码
            if _, ok := err.(*validator.InvalidValidationError); ok {
                ctx.StatusCode(iris.StatusInternalServerError)
                ctx.WriteString(err.Error())
                return
            }

            ctx.StatusCode(iris.StatusBadRequest)
            for _, err := range err.(validator.ValidationErrors) {
                fmt.Println()
                fmt.Println(err.Namespace())
                fmt.Println(err.Field())
                fmt.Println(err.StructNamespace())
                fmt.Println(err.StructField())
                fmt.Println(err.Tag())
                fmt.Println(err.ActualTag())
                fmt.Println(err.Kind())
                fmt.Println(err.Type())
                fmt.Println(err.Value())
                fmt.Println(err.Param())
                fmt.Println()
            }

            return
        }

        // [将user存入数据库...]
    })

    app.Listen(":8080")
}

func UserStructLevelValidation(sl validator.StructLevel) {
    user := sl.Current().Interface().(User)

    if len(user.FirstName) == 0 && len(user.LastName) == 0 {
        sl.ReportError(user.FirstName, "FirstName", "fname", "fnameorlname", "")
        sl.ReportError(user.LastName, "LastName", "lname", "fnameorlname", "")
    }
}
Example request of JSON form:

{
    "fname": "",
    "lname": "",
    "age": 45,
    "email": "mail@example.com",
    "favColor": "#000",
    "addresses": [{
        "street": "Eavesdown Docks",
        "planet": "Persphone",
        "phone": "none",
        "city": "Unknown"
    }]
}
```

JSON格式的请求示例:

```json
{
    "fname": "",
    "lname": "",
    "age": 45,
    "email": "mail@example.com",
    "favColor": "#000",
    "addresses": [{
        "street": "Eavesdown Docks",
        "planet": "Persphone",
        "phone": "none",
        "city": "Unknown"
    }]
}
```

可以在[这里](https://github.com/kataras/iris/tree/master/_examples/request-body/read-json-struct-validation/main.go)找更多的例子
