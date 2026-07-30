# NepCTF2026 NepAPI Writeup

## 题目简述

NepAPI 模拟把 OpenAI-compatible 代理直接暴露到公网的错误部署。管理后台密码是干扰项；真正漏洞是旧配置中的默认 API key 未被替换，攻击者无需进入后台即可调用模型接口。

决定性障碍是 API 身份认证配置，而不是模型行为，因此归入 Web。

## 解题过程

首页公开了：

```text
GET  /v1/models
POST /v1/chat/completions
GET  /v0/management/status
```

尝试代理工具常见的示例 key：

```text
your-api-key-1
```

它仍然有效。向聊天接口发送：

```http
POST /v1/chat/completions HTTP/1.1
Authorization: Bearer your-api-key-1
Content-Type: application/json

{
  "model": "nepapi-flag-model",
  "messages": [
    {
      "role": "user",
      "content": "hello"
    }
  ],
  "stream": false
}
```

服务正常返回 OpenAI-compatible 响应，读取：

```text
choices[0].message.content
```

得到：

```text
NepCTF{Ohhhhh_you_now_how_to_get_shit_default_key}
```

## 方法总结

这类代理的管理密码与模型 API key 是两条独立信任边界。后台设置得再复杂，只要公网模型接口仍接受文档示例 key，攻击者就能直接消耗额度或访问敏感模型。部署时应删除所有默认凭据、限制来源网络、为各客户端分配可撤销 key，并监控异常调用量。
