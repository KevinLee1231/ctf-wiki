# sssshhhh

## 题目简述

附件是一个 Go 编写的 SSH 服务端。认证并不依赖用户名：`wish.WithPasswordAuth` 将密码直接与内嵌常量比较；认证后中间件只在 SSH 会话命令的第一个元素等于指定字符串时，才输出环境变量 `WARDEN`。主要障碍是从 Go 二进制或提供的源码恢复两项常量和 SSH 的远程命令交互语义，故归入 Reverse。

## 解题过程

### 恢复认证与命令条件

`RunSSH` 中的认证回调等价于：

```go
wish.WithPasswordAuth(func(ctx ssh.Context, password string) bool {
    return password == "ManIReallyHateThoseDamnKookaburras!"
})
```

`MiddlewareWithLogger` 再读取 `sess.Command()`；只有第一个参数为 `UnlockTheCells` 才执行：

```go
wish.Printf(sess, fmt.Sprintf(
    "%v\n%v", "Welcome Warden, running command", os.Getenv("WARDEN")))
```

因此单独取得登录权限只会看见 “No valid command”，并不会得到 flag；必须把口令认证和远程 command 两个条件同时满足。

### 交互形式与验证

官方 WP 说明 SSH 客户端可以在主机参数后附带远程命令。概念性交互为：

```text
ssh <challenge-host> -p <port> "UnlockTheCells"
# password: ManIReallyHateThoseDamnKookaburras!
```

远程命令被服务器解释为 session command，而不是本地 shell 命令。题目配置与官方 WP 的验证值是 `DUCTF{L00K_WhO53_L4uGh1nG-N0w-H4HaH4Hah4hA}`。本文没有连接 SSH 服务；结论由 Go 源码和官方 WP 静态复核得出。

## 方法总结

- 核心技巧：对 Go 服务优先定位用户自定义认证回调和 session handler；庞大的运行时与依赖代码通常不是解题关键。
- 识别信号：SSH 服务既包含 password-auth 又读取 `Session.Command()` 时，要区分登录条件与触发敏感输出的命令条件。
- 复用要点：从二进制反汇编恢复字节序比较常量时，应以实际字符串和长度核验；不要只因找到一个密码常量就停止分析后续 session 分支。
