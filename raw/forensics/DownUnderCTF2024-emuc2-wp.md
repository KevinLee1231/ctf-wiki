# emuc2

## 题目简述

题目给出 PCAP 与 TLS key log。虽然背景是自研恶意软件/C2，但解题核心是从已捕获的 TLS 流量恢复 API 凭据和历史环境变量，再据此重建 JWT；不需要分析恶意样本或其持久化/C2 协议实现。因此按决定性障碍归入 Forensics。

源码确认 API 使用 HS512 JWT，只有 `subject_id == 1` 能访问 `/api/flag`；C2 把客户端环境变量保存为可列举的 `loot` 文件，其中一份历史记录泄漏了签名密钥。

## 解题过程

在 Wireshark 的 TLS 设置中载入提供的 `sslkeylogfile.txt`，随后过滤 HTTP/2 请求。解密后的流量显示同一服务上的登录、`/api/env` 读取/上传和 `/api/flag` 请求。登录请求给出一组低权限 C2 凭据；源码中对应的用户名为 `jooospeh`，密码为 `n3v3r-g0nna-g1v3-th3-b1rds-up`。

用该会话读取 `/api/env` 获取随机文件名列表，并逐个访问 `/api/env/<id>`。每个有效历史文件首行是 UTC 时间戳，后续为环境变量。按时间排序后，唯一的 2023 年记录最早，其中含有：

```text
JWT_SECRET=<泄漏的长随机字符串>
```

这不是猜测 JWT 密钥：题目源码的 `secret.txt` 正是由同一个 `JWT_SECRET` 生成，验证器固定为 `Algorithm::HS512`。因此用泄漏字符串作为 HMAC-SHA512 密钥，构造如下语义的 JWT：

```json
{
  "subject_id": 1,
  "exp": "未来的 Unix 时间戳"
}
```

将 token 以 `Authorization: Bearer <token>` 请求 `/api/flag` 即可通过角色检查。源码中的 flag 为：

```text
DUCTF{pǝʇɔǝɟuᴉ_sᴉ_ǝlᴉɟ_dᴉz_ǝɥʇ_oʇ_pɹoʍssɐd_ǝɥʇ}
```

## 方法总结

TLS key log 能将 PCAP 中本不可读的 HTTPS/HTTP2 还原为应用层证据。分析时应按请求时序重建“登录、枚举、读取单项、提权”的 API 关系，而不是只搜索 flag 字符串。C2 或日志系统绝不能收集、保存并向低权限用户公开环境变量，因为其中常含令牌和签名密钥。
