# secure email attachments

## 题目简述

题目给出一个 Go/Gin 静态文件服务，并要求读取 `/etc/flag.txt`。服务先拒绝原始 URL 路径中的字面量 `..`，随后删除所有 `/attachments` 子串，再执行 `filepath.Clean()` 和 `filepath.Join()`。过滤与规范化的顺序错误，使攻击者能够在过滤后“合成”出父目录跳转片段。

## 解题过程

核心路由如下：

```go
p := c.Param("path")
if strings.Contains(p, "..") {
    c.AbortWithStatus(400)
    return
}

cleanPath := filepath.Join(
    "./attachments",
    filepath.Clean(strings.ReplaceAll(p, "/attachments", "")),
)
http.ServeFile(c.Writer, c.Request, cleanPath)
```

直接提交 `../../etc/flag.txt` 会被第一层检查拦截。但可以把每对点号拆开，让原始字符串不含连续的 `..`，再借助 `ReplaceAll()` 删除中间片段。例如官方 payload：

```http
GET /attachments./attachments./attachments/./attachments./attachments/etc/flag.txt HTTP/1.1
Host: <challenge-host>
```

原始路径里每个点之间都被 `/attachments` 隔开，因此通过 `strings.Contains(p, "..")`。删除 `/attachments` 后，剩余字符重新拼成父目录跳转；`filepath.Clean()` 只负责规范化 `.`、`..` 和分隔符，并不会把攻击路径限制在基准目录中。`filepath.Join("./attachments", ...)` 继续折叠父目录，最终解析到 `/etc/flag.txt`，`http.ServeFile()` 随即返回文件内容。

## 方法总结

- 核心技巧：利用“先过滤、后变换”造成的路径重写差异，在过滤通过后合成 `..`。
- 识别信号：用户输入先做字面量检查，随后还有 `ReplaceAll`、URL 解码、规范化或路径拼接等会改变字符串语义的操作。
- 复用要点：安全校验必须针对最终规范化路径，并用可靠的目录边界检查确认结果仍位于允许根目录；`Clean` 不是净化器，黑名单也不能替代 canonicalization 后的 containment 验证。
