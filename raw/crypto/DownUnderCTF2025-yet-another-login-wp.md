# yet another login

## 题目简述

注册接口为用户名生成消息 `b"user=" + username`，并返回 Paillier 加密的

$$
h=\operatorname{SHA256}(secret\parallel msg).
$$

登录时服务端解密 token，但只比较解密整数的低 32 字节：`sha256(secret + msg).digest() == long_to_bytes(decrypt(mac))[-32:]`。公开的 Paillier 模数 $n$ 与 $g=n+1$ 使攻击者能同态地加、减和乘明文；验证成败则成为一个 bit oracle。恢复一条普通用户的完整 SHA-256 状态后，再利用 `SHA256(secret || msg)` 的长度扩展即可把末尾用户改成 `admin`。

## 解题过程

### 从注册 token 恢复原始 hash

先注册不含 `admin` 的用户名，例如 `user`，得到消息 $m=\texttt{user=user}$ 和密文 $c=E(h)$。Paillier 在 $g=n+1$ 下满足：

$$
E(a)E(b)=E(a+b),\qquad E(a)^k=E(ka),\qquad E(a)g^{-t}=E(a-t).
$$

设 `rec` 已恢复 $h$ 的低 $i$ bit。为测试第 $i$ bit，构造

$$
w=E\left(h+(h-rec)2^{255-i}\right).
$$

代码形式为：

```python
def add_plain(c, a):
    return c * pow(g, a, n * n) % (n * n)

def multiply_plain(c, k):
    return pow(c, k, n * n)

for i in range(256):
    cleared = add_plain(c, -rec)
    probe = c * multiply_plain(cleared, 1 << (255 - i)) % (n * n)
    if not login_oracle(b"user=user", long_to_bytes(probe)):
        rec |= 1 << i
```

因为 `rec` 的低 $i$ bit 正确，$(h-rec)2^{255-i}\bmod2^{256}$ 只保留待测 bit 对应的 $2^{255}$。待测 bit 为 0 时，解密结果的低 256 bit 仍为 $h$，登录成功；待测 bit 为 1 时最高位被翻转，验证失败。循环从低到高恢复完整 256 bit `rec`，即 $\operatorname{SHA256}(secret\parallel\texttt{user=user})$。

### SHA-256 长度扩展为 admin 消息

源码明确 `secret` 长度为 16 字节，且 MAC 错误地使用前缀拼接 `SHA256(secret || msg)`，不是 HMAC。随附的 `hlextend.py` 可把已恢复的 hash 当作 SHA-256 内部状态，生成：

$$
m'=m\parallel\operatorname{pad}(16+|m|)\parallel\texttt{user=admin},
$$

及其正确摘要 $h'=\operatorname{SHA256}(secret\parallel m')$。攻击脚本中的对应调用为：

```python
hle = hlextend.sha256()
forged_msg = hle.extend(
    b"user=admin", b"user=user", 16, long_to_bytes(rec).hex()
)
forged_hash = bytes.fromhex(hle.hexdigest())
assert forged_msg.rpartition(b"user=")[2] == b"admin"
```

服务端使用 `rpartition(b"user=")` 取用户名，因此 SHA-256 padding 中的二进制字节不会妨碍它取最后一个 `user=admin`。

### 重新加密并登录

无需 Paillier 私钥即可生成 $E(h')$。取随机数 $r=1$ 就有合法密文 $g^{h'}\bmod n^2$：

```python
forged_mac = long_to_bytes(pow(n + 1, bytes_to_long(forged_hash), n * n))
login_oracle(forged_msg, forged_mac)
```

该 token 解密后的低 32 字节等于 $h'$，而消息最后一个用户字段是 `admin`，故能触发 flag 分支。

### 验证

官方两份 solver 都先做 256 次登录 oracle 查询，再使用长度扩展并以 $r=1$ 重加密。题目配置给出的验证材料为 `DUCTF{now_that_youve_logged_in_its_time_to_lock_in}`。本归档未连接服务；恢复公式、oracle 极性和消息格式均已与 `HINTS.md`、源码和官方 solver 静态核对。

## 方法总结

- 核心技巧：Paillier 的加法同态配合“只比较低 256 bit”的错误校验，能把登录成功/失败变成逐位明文 oracle；随后对前缀 MAC 做 SHA-256 长度扩展。
- 识别信号：加密 MAC 若没有完整比较、把大整数截断为 hash 长度，或允许攻击者提交同态变形后的密文时，应先寻找可控制的 carry/bit 翻转，再考虑直接解密。
- 复用要点：长度扩展只适用于 `Hash(secret || data)` 这类 Merkle--Damgård 前缀 MAC，且必须知道或枚举 secret 长度。构造扩展消息时要保留二进制 padding；不要把它当 UTF-8 文本处理。
