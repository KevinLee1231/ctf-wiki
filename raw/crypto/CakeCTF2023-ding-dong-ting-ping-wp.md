# ding-dong-ting-ping

## 题目简述

服务提供注册与登录两个接口。注册时，明文用户名中只要出现 `root` 就会被拒绝；合法用户名则被拼成：

```text
PREFIX|user=<username>|<timestamp>
```

部署配置把 `PREFIX` 设为 17 字节的 `TING~DING~DONG~XD`。cookie 使用固定随机 IV 和 AES-ECB 自行拼出一条链：

$$
C_0=IV,\qquad
C_i=E_k\bigl(P_i\oplus MD5(C_{i-1})\bigr).
$$

解密时执行对应逆变换：

$$
P_i=D_k(C_i)\oplus MD5(C_{i-1}).
$$

这不是带认证的加密结构。固定 IV 还会让相同的明文前缀产生相同密文块，因此可以从两份合法 cookie 中裁剪和重排块，伪造出 `user=root`。决定性障碍是自定义密码链的可塑性，归入 `crypto`。

## 解题过程

### 对齐前两个明文块

选取 9 字节用户名 `aaaaaaaaa`。由于 `PREFIX|user=` 长 23 字节，加上 9 字节用户名后正好是 32 字节，前两个明文块固定为：

```text
P1 = TING~DING~DONG~X
P2 = D|user=aaaaaaaaa
```

注册一次并把返回值按 16 字节分成 $C_0,C_1,C_2,\ldots$，计算：

$$
h_1=MD5(C_1),\qquad h_2=MD5(C_2).
$$

### 构造将被错位解密的块

希望伪造 cookie 的第二个明文块为：

```text
T = <1 byte>|user=root|aaaa
```

它恰好为 16 字节。构造一个 16 字节用户名后缀：

$$
Q=h_1\oplus T\oplus h_2.
$$

再次注册用户名 `aaaaaaaaa || Q`。它的前两块明文仍与第一次完全相同，因此固定 IV 下仍会得到同样的 $C_1,C_2$；而 $Q$ 恰好占据第三个完整明文块，所以第二份 cookie 的第三密文块为：

$$
C_3'=E_k(Q\oplus h_2)=E_k(T\oplus h_1).
$$

现在保留第一份 cookie 的 `C0 || C1`，跳过其 `C2`，从第二份 cookie 的 `C3'` 开始拼接：

```python
attack_cookie = cookie[:32] + payload_cookie[48:]
```

服务解密拼接结果时，会把 $C_3'$ 当成第二个密文块：

$$
D_k(C_3')\oplus MD5(C_1)
=(T\oplus h_1)\oplus h_1
=T.
$$

第二份 cookie 后面的密文块仍按其原有链关系恢复时间戳，所以最终明文结构完整。`T` 的第一个字节需要补上 `PREFIX` 的最后一个字符；部署值中该字节是 `D`（十进制 68）。官方脚本不知道环境中的具体前缀，便枚举 0 到 255，直到响应出现 `Hi, root!`。

以下是与官方 `solution/solver.py` 一致的交互核心；`sock` 表示已经连接到服务的 `ptrlib.Socket`：

```python
from base64 import b64decode, b64encode
from hashlib import md5

xor = lambda a, b: bytes(x ^ y for x, y in zip(a, b))
username = b"a" * 9

sock.sendlineafter(": ", "1")
sock.sendlineafter("username(base64): ", b64encode(username))
cookie = b64decode(sock.recvlineafter("cookie => "))
blocks = [cookie[i:i + 16] for i in range(0, len(cookie), 16)]

h1 = md5(blocks[1]).digest()
h2 = md5(blocks[2]).digest()

for candidate in range(256):
    target = bytes([candidate]) + b"|user=root|aaaa"
    suffix = xor(xor(h1, target), h2)

    sock.sendlineafter(": ", "1")
    sock.sendlineafter("username(base64): ", b64encode(username + suffix))
    payload_cookie = b64decode(sock.recvlineafter("cookie => "))

    attack_cookie = cookie[:32] + payload_cookie[48:]
    sock.sendlineafter(": ", "2")
    sock.sendlineafter("cookie: ", b64encode(attack_cookie))

    if b"Hi, root!" in sock.recvline():
        break
```

伪造成功后服务输出：

```text
CakeCTF{dongdingdongding-dingdong-dongdingdong-ding}
```

## 方法总结

- 核心技巧：利用固定 IV、可计算的 `MD5(C_i)` 链和密文块错位拼接，把第二份 cookie 的第三块变成第一份 cookie 的第二块。
- 识别信号：服务把 ECB 当底层原语自制链式模式，解密后直接解析权限字段，却没有 MAC 或 AEAD 标签；相同前缀还能稳定复用密文块。
- 复用要点：先精确画出明文与密文块边界，再写每个被拼接块在新前驱下的解密公式。这里不是简单翻转某个原密文块；关键是先通过注册接口生成满足目标代数关系的新块，再删除中间块制造状态错位。
