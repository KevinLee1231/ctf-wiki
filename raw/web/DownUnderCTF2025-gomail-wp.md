# gomail

## 题目简述

这是一个 Go 邮件站。`POST /login` 用服务器随机生成的密钥签发 `X-Auth-Token`，`GET /emails` 只在 token 的 `IsAdmin` 为真时返回管理员邮箱 `mc-fat@monke.zip` 中的邮件和 flag。token 使用 HMAC-SHA256 签名，因此不能伪造签名；漏洞在于服务器会替攻击者签名一份长度字段溢出的序列化 claims。

claims 的二进制布局为 `uint16(email 长度) || email || int64(Expiry) || 1 字节 IsAdmin`。序列化时把 Go 的 `int` 截断成 `uint16` 写入长度，却仍将完整 email 写入缓冲区。只要登录一个长度超过 $2^{16}$ 的、且不是管理员邮箱的 email，就能让签发 token 时写入的长度与反序列化时消费的字节数不同。

## 解题过程

### 关键观察

管理员密码无法获得：错误密码会被降级为 guest。不过未知 email 不会被替换，仍会作为非管理员 claims 交给 `Encode`。问题代码如下：

```go
func (ss *SessionSerializer) writeLength(l int) {
    el := uint16(l)
    binary.LittleEndian.PutUint16(bs, el)
    ss.buf.Write(bs)
}

func (ss *SessionSerializer) writeEmail(email string) {
    ss.writeLength(len(email))
    ss.buf.WriteString(email)
}
```

而解码端只依照那两个字节取 email，随后连续读取 8 字节 expiry 和 1 字节布尔值。HMAC 覆盖的是这份错误布局，但它由服务端自己生成，所以校验仍会通过。

### 构造 token

目标是使截断后的 email 长度恰好为管理员邮箱的 16 字节。把开头设为该邮箱后，在其后塞入 8 个 `z` 和一个 `t`，再补足到总长度 $2^{16}+16$：

```python
admin = 'mc-fat@monke.zip'
email = admin + 'z' * 8 + 't' + 'A' * ((1 << 16) - 9)
payload = {'email': email, 'password': 'anything'}
```

这里尾部填充的长度使 `uint16(len(email)) == 16`。反序列化结果因而是：

1. `Email` 只读开头的 `mc-fat@monke.zip`；
2. 接下来的 `zzzzzzzz` 被按 little-endian `int64` 解为一个很大的正数，作为远未来 expiry；
3. 接下来的 `t` 被 `readIsAdmin` 解释为真。

将 `/login` 响应的 token 放入邮件接口的 header 即可：

```http
GET /emails HTTP/1.1
X-Auth-Token: <login 返回的 token>
```

JSON 解码会额外转义某些字符并改变实际字节数，因此填充应采用普通 ASCII；同时请求体必须低于程序设置的 1 MiB 上限。

### 验证

返回的邮件数组应来自管理员邮箱，而非 guest 的诱饵邮件。官方源码中的 flag 为：

```text
DUCTF{g0v3rFloW_2_mY_eM41L5!}
```

## 方法总结

- 核心技巧：利用长度前缀截断造成的结构错位，让可信签名覆盖攻击者选择的反序列化 claims。
- 识别信号：自定义 token 在不同宽度的整数之间转换，且写入长度和写入数据使用的长度不一致时，应画出精确的字节布局。
- 复用要点：签名只能保证字节未被传输后篡改，不能修复签名方序列化出的歧义。长度必须在转换前显式拒绝超过 `uint16` 上限的输入。
