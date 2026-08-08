# MiniLCTF2022 Checkin Writeup

## 题目简述

应用把一段 JSON 会话数据按 PKCS#7 填充后用 AES-CBC 加密，并将 `IV || ciphertext` 做 Base64、URL 编码后放入 `token` Cookie。普通用户的首块明文包含 `"Name":"guest"`，而服务端对解密或填充失败与有效请求给出可区分响应，因此既存在 CBC 位翻转，也存在 Padding Oracle。

## 解题过程

已知明文结构为：

```json
{"Name":"guest","CreateAt":1651331029,"IP":"127.0.0.1"}
```

CBC 首块解密满足 $P_1=D_K(C_1)\oplus IV$。因此只要修改 IV，不必知道 AES 密钥便能定向修改首块明文。`guest` 位于首块偏移 9，令

$$
IV'[9:14]=IV[9:14]\oplus\texttt{guest}\oplus\texttt{admin},
$$

其余 IV 字节和密文保持不变：

```python
import base64
import urllib.parse

raw = base64.b64decode(urllib.parse.unquote(token))
iv, ciphertext = bytearray(raw[:16]), raw[16:]

old = b"guest"
new = b"admin"
for i, (a, b) in enumerate(zip(old, new), start=9):
    iv[i] ^= a ^ b

forged = urllib.parse.quote(base64.b64encode(bytes(iv) + ciphertext))
```

将 `forged` 作为新的 `token` Cookie 请求 `/home`，首块就会被解析为管理员身份。若初始明文未知，也可利用响应差异逐字节构造合法的 `01`、`02 02` 等填充，从后向前恢复每一块的中间值 $D_K(C_i)$，再与前一密文块异或得到完整 JSON；官方脚本同时给出了这条通用 Padding Oracle 路线。

## 方法总结

CBC 加密只提供保密性，不提供完整性。攻击者可通过改动前一密文块或 IV 精确翻转后一明文块；若服务还泄漏填充是否合法，则可进一步解密任意块。此题已有已知明文和等长替换，直接位翻转最短。修复应使用带认证的 AEAD 模式，或至少先验证强 MAC，再统一处理所有解密错误。
