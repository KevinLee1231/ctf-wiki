# FileSystem

## 题目简述

题目是一个 Go 文件服务器。根路由通过 `http.FileServer(http.Dir("./"))` 暴露当前目录，但程序又为 `/there_may_be_a_flag` 单独注册了拒绝访问的处理器。需要利用 Go 标准库对 `CONNECT` 请求路径的特殊处理，让外层 `ServeMux` 选中根路由，再由内层文件服务器把畸形路径规范化为真实文件名。

## 解题过程

服务端逻辑可整理为：

```go
func fileHandler(w http.ResponseWriter, r *http.Request) {
    http.FileServer(http.Dir("./")).ServeHTTP(w, r)
}

func main() {
    http.HandleFunc("/", fileHandler)
    http.HandleFunc("/there_may_be_a_flag", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("No! You can't see the flag!"))
    })
    log.Fatal(http.ListenAndServe(":8889", nil))
}
```

普通请求 `/there_may_be_a_flag` 会精确匹配第二个处理器，因此只能收到拒绝文本。这里有两个可组合的标准库行为：

1. 旧版 Go `net/http` 的 `ServeMux` 对普通方法会先执行 `cleanPath`，但对 `CONNECT` 请求不做同样的路径规范化。
2. `http.HandleFunc` 没有限定请求方法，所以注册的路径同样会接收 `CONNECT`；而进入根处理器后，`http.FileServer` 又会在查找磁盘文件前清理路径。

因此发送路径为 `//there_may_be_a_flag` 的 `CONNECT` 请求。外层 `ServeMux` 看到未经清理的双斜杠路径，它不等于受保护的精确路由，只能落入 `/` 根路由；`FileServer` 随后把双斜杠清理为单斜杠，并读取真实文件：

```bash
curl -X CONNECT --path-as-is \
  http://filesystem.hgame.homeboyc.cn//there_may_be_a_flag
```

`--path-as-is` 很重要，否则 curl 可能在发送前自行规范化路径。也可以考虑 `/../there_may_be_a_flag`，但官方部署前还有 Nginx 反向代理，该形式会先被代理层拦截；双斜杠才是适用于题目环境的预期构造。响应正文即为 flag 文件内容。

## 方法总结

漏洞来自同一请求在两层组件中的路径解释不同：`ServeMux` 用未规范化的 `CONNECT` 路径选路由，`FileServer` 却用规范化后的路径访问文件。遇到反向代理、路由器和静态文件服务串联的应用时，应逐层确认方法、URL 编码和路径清理顺序；只在外层注册一个“禁止访问”的精确路由，并不能保护同目录下的敏感文件。
