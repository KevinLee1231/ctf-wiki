# JWTF

## 题目简述

应用公开一份 JWT Revocation List，其中包含合法签名的管理员 token。`/flag` 先移除 `=` 和首尾空白，再按 token 的原始字符串是否出现在撤销列表中进行拦截，之后才用 HS256 验签并检查 `admin` 字段。

撤销逻辑比较的是 Base64URL 文本，而验签逻辑比较解码后的签名字节；非规范 Base64URL 编码可让两者产生分歧。

## 解题过程

从 `/jrl` 取得撤销的管理员 token。HS256 签名为 32 字节，Base64URL 无填充编码长度为 43 个字符；末字符只有高 4 位属于有效数据，低 2 位是未使用填充位。改变这些未使用位后，文本会变化，但解码出的 32 字节签名完全相同。

官方示例签名末尾为：

```text
...Mj_UnvZk
```

把最后一个 `k` 改为同一有效高位组中的 `l`，可验证：

```python
import base64

def b64url_decode(s):
    return base64.urlsafe_b64decode(s + "=" * (-len(s) % 4))

assert b64url_decode(sig[:-1] + "k") == b64url_decode(sig[:-1] + "l")
```

将修改后的完整 token 放入 `session` Cookie。它不再与 JRL 中的字符串相等，却仍能通过 HS256 验签，payload 中的 `admin: true` 生效并返回：

```text
byuctf{idk_if_this_means_anything_but_maybe_its_useful_somewhere_97ba5a70d94d}
```

修改 header 或 payload 段会改变签名输入，不能通过；增加 `=` 或空白也会被路由先规范化掉。

## 方法总结

- 核心技巧：利用 Base64URL 末尾未使用位制造“不同文本、相同字节”的 JWT 签名编码，绕过字符串型撤销列表。
- 识别信号：安全判断若对编码文本做精确比较，而后续密码验证先解码，应检查非规范编码、填充和大小写等表示层差异。
- 复用要点：撤销表应基于稳定标识如 `jti`，或先做严格规范化并拒绝非规范编码；不能只比较外部 token 字符串。
