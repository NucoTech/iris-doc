# 会话数据库(Sessions Database)

有时, 您需要一个后端存储, 比如redis, 它会在服务器重启时保存你的会话数据并横向扩展

注册会话数据库可以单次调用 `session.UseDatabase (database)` 来完成

Iris为[redis](https://redis.io/)、[badger](https://github.com/dgraph-io/badger)和[boltdb](https://github.com/kataras/iris/wiki/github.com/etcd-io/bbolt)实现了内置会话数据库。您需要导入[iris/sessions/sessiondb](https://github.com/kataras/iris/tree/master/sessions/sessiondb)子包实现

1. 导入 `gitbub.com/kataras/iris/sessions/sessiondb/redis或boltdb或badger`
2. 初始化数据库
3. 注册

例如, 注册redis会话数据库:

```go
import (
    "github.com/kataras/iris/v12"
    "github.com/kataras/iris/v12/sessions"

    // 1. 引入会话数据库.
    "github.com/kataras/iris/v12/sessions/sessiondb/redis"
)

// 2. 初始化数据库
// 这些是默认值, 您可以根据正在运行的redis数据库进行修改
db := redis.New(redis.Config{
    Network:   "tcp",
    Addr:      "127.0.0.1:6379",
    Timeout:   time.Duration(30) * time.Second,
    MaxActive: 10,
    Password:  "",
    Database:  "",
    Prefix:    "",
    Delim:     "-",
    Driver:    redis.Redigo(), // 可用redis.Radix()代替 
})

// 可以通过选项的方式来配置驱动器
// driver := redis.Redigo()
// driver.MaxIdle = ...
// driver.IdleTimeout = ...
// driver.Wait = ...
// redis.Config {Driver: driver}

// 按下control+C/cmd+C关闭连接
iris.RegisterOnInterrupt(func() {
    db.Close()
})

sess := sessions.New(sessions.Config{
    Cookie:       "sessionscookieid",
    AllowReclaim: true,
})

// 3. 注册
sess.UseDatabase(db)

// [...]
```

boltdb

```go
import "os"
import "github.com/kataras/iris/v12/sessions/sessiondb/boltdb"

db, err := boltdb.New("./sessions.db", os.FileMode(0750))
```

badger

```go
import "github.com/kataras/iris/v12/sessions/sessiondb/badger"

db, err := badger.New("./data")
```
