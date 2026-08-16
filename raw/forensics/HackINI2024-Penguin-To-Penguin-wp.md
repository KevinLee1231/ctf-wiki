# Penguin To Penguin

## 题目简述

附件是一份包含大量普通网络流量的 PCAPNG，要求找出通信端点暴露的公网 IP。抓包中存在发往 UDP 19302 的 STUN Binding 请求与响应；STUN 的 `XOR-MAPPED-ADDRESS` 属性记录 NAT 映射后的公网地址。

## 解题过程

### 定位 STUN Binding Response

在 Wireshark 中使用显示过滤器：

```text
stun
```

选中 Binding Success Response，展开：

```text
STUN Message
└── XOR-MAPPED-ADDRESS
```

该属性不会直接存储 IP，而是将地址与 STUN magic cookie `0x2112A442` 异或。附件中的 IPv4 异或值为：

```text
bb 6b f5 99
```

逐字节还原：

```text
bb ^ 21 = 9a = 154
6b ^ 12 = 79 = 121
f5 ^ a4 = 51 = 81
99 ^ 42 = db = 219
```

因此公网 IP 为：

```text
154.121.81.219
```

Wireshark 会自动完成这一步并在字段值中显示解码后的地址。最终 flag：

```text
shellmates{154.121.81.219}
```

## 方法总结

- 核心技巧：从 STUN Binding Response 的 `XOR-MAPPED-ADDRESS` 恢复 NAT 映射后的公网 IP。
- 识别信号：UDP 19302、STUN magic cookie `0x2112A442` 和 Binding Success Response 是常见定位点。
- 复用要点：IPv4 地址与 32 位 cookie 异或，端口则与 cookie 的高 16 位异或；不要把抓包中的内网源地址误当公网映射地址。
