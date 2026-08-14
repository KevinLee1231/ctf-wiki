# bi0sCTF 2022 - insane key

## 题目简述

附件是一把被大量 `?` 破坏的 OpenSSH RSA 私钥。目标不是用它登录，而是尽可能精确地恢复原文件；题目以 `sha256sum key` 的结果作为 flag，正确摘要以 `05bc9` 开头。

OpenSSH 新式私钥在 Base64 解码后包含 `openssh-key-v1\0` 魔数、加密/KDF 描述、公钥块和私钥块。私钥块内部依次保存两个相同的 check integer、密钥类型、$n,e,d,iqmp,p,q$、注释和顺序填充。完整格式可参见 [OpenSSH private key binary format](https://dnaeon.github.io/openssh-private-key-binary-format/)；本题实际用到的要点是字段采用 4 字节大端长度前缀，而 `iqmp` 的含义为 $q^{-1}\bmod p$。

仓库中的 `analysis.docx` 只是对同一段十六进制数据做颜色标注，没有嵌入图片；有效信息已经全部转写到下文，因此无需保留截图。

## 解题过程

### 解析残缺字段

先暂时把未知 Base64 字符 `?` 替换为 `/`。`/` 对应 Base64 数值 63，因此未知区会在解码后表现为连续的 `0xff`，便于在十六进制中辨认字段边界：

```python
from base64 import b64decode

raw = open("key", "rb").read().replace(b"?", b"/")
body = b"".join(
    line for line in raw.splitlines()
    if not line.startswith(b"-----")
)
decoded = b64decode(body)
open("damaged.bin", "wb").write(decoded)
```

按 OpenSSH 字段长度解析后，可以可靠取得：

- 公钥指数 $e=65537$；
- 两段相互重叠的模数片段，合并后得到完整 $n$；
- 完整的 $u=iqmp$；
- $p$ 的高位；
- $q$ 的高位估计、中间片段和低位片段。

由于 $n=pq$，用已知的 `p_upper` 作整数除法可以估计 $q$ 的高位：

```python
q_upper = int(hex(n // p_upper)[:len(hex(p_upper)) - 2], 16)
```

本题得到：

```python
q_upper  = 0xc597ff4fc78fabff87fd021ee21d02e2b2f726cbe12
q_middle = 0xc570613da06c478
q_lower  = 0x691738dc8996d
```

### 利用 iqmp 建立模方程

令 $u=iqmp=q^{-1}\bmod p$，则

$$
uq\equiv1\pmod p.
$$

两边乘以 $q$，并使用 $n=pq$，得到：

$$
uq^2-q\equiv0\pmod n.
$$

把 $q$ 中的两段未知区分别记为 $x_0,x_1$。根据字段在十六进制串中的位置，可写成：

```python
PR.<x0, x1> = PolynomialRing(Zmod(n), 2)

qq = (q_upper  * 2^(4 * 341)
      + q_middle * 2^(4 * 116)
      + q_lower
      + 2^52  * x0
      + 2^524 * x1)

f = u * qq^2 - qq
```

未知块满足 $x_0<2^{412}$、$x_1<2^{840}$。对 $f(x_0,x_1)\equiv0\pmod n$ 使用二元 Coppersmith：构造 $f^k n^{m-k}$ 的移位多项式，按变量界缩放系数矩阵，LLL 约化后从短向量重建整数多项式，再用 Jacobian/Newton 法求公共小根。官方参数为 `m=3, d=2`：

```python
x0, x1 = coppersmith(
    f,
    bounds=(2^412, 2^840),
    m=3,
    d=2,
)
q = int(qq(x0, x1))
assert is_prime(q) and n % q == 0
```

恢复出的 $q$ 为：

```text
1860400960117464949532416004639466629994076821767309538826515112720671132102034738577938121013868915703939452009570308375925288112735480787384780575361241826457665569291257342639256426234197438821880680008965742321854591919619063729626119862840545588909676241323030007484307455645544766439964269237529144790202822411163427130628861671228602354772689245481045576816266142485189200125194387337260813139712001196664255796834077354270127893071327103533571746413713773
```

### 重建私钥文件

其余 RSA 参数可以直接计算：

```python
from Crypto.PublicKey import RSA
from math import lcm

p = n // q
d = pow(e, -1, lcm(p - 1, q - 1))
assert p * q == n

pem = RSA.construct((n, e, d, p, q)).export_key("PEM")
open("key.pem", "wb").write(pem)
```

用 `ssh-keygen` 在空口令下重写为 OpenSSH 私钥格式。为了避免交互式输入，可以显式给出新旧空口令。原残缺文件恰好完整保留了两份 check integer：Base64 片段 `eIwxH26MMR9u` 解码后对应大端整数 `0x8c311f6e` 重复两次。不同 OpenSSH 版本重写时会随机生成新值，而题目要求文件级 SHA-256 完全一致，所以必须把这 8 字节改回原值。

先把 `RSA.construct` 生成的 PEM 容器转换成无口令的 OpenSSH 容器：

```bash
chmod 600 key.pem
ssh-keygen -p -P "" -N "" -o -f key.pem
```

OpenSSH 容器中的私钥字符串位于三段 cipher/KDF 字符串、密钥数量和公钥字符串之后。下面的代码只替换私钥字符串开头的两份 check integer，并按原文件的 70 字符行宽和 LF 换行重新封装：

```python
import base64
import struct
import textwrap

lines = open("key.pem", "r", newline="").read().splitlines()
body = "".join(line for line in lines if not line.startswith("-----"))
raw = bytearray(base64.b64decode(body))

offset = len(b"openssh-key-v1\0")

def skip_string(buf, pos):
    size = struct.unpack(">I", buf[pos:pos + 4])[0]
    return pos + 4 + size

for _ in range(3):                 # ciphername、kdfname、kdfoptions
    offset = skip_string(raw, offset)

key_count = struct.unpack(">I", raw[offset:offset + 4])[0]
offset += 4
for _ in range(key_count):         # public key strings
    offset = skip_string(raw, offset)

private_size = struct.unpack(">I", raw[offset:offset + 4])[0]
private_offset = offset + 4
assert private_size > 8
raw[private_offset:private_offset + 8] = struct.pack(
    ">II", 0x8C311F6E, 0x8C311F6E
)

encoded = base64.b64encode(raw).decode()
armored = (
    "-----BEGIN OPENSSH PRIVATE KEY-----\n"
    + "\n".join(textwrap.wrap(encoded, 70))
    + "\n-----END OPENSSH PRIVATE KEY-----\n"
)
with open("key", "w", newline="\n") as out:
    out.write(armored)
```

原私钥的 comment 为空，末尾顺序填充为 `01 02 03`；重写后也应检查这两项。最终校验：

```bash
sha256sum key
```

得到：

```text
05bc9eca84c44570670df15d29a37d78ffe9813073ca41d2147b14a06e9d96a1
```

因此 flag 为：

```text
bi0sctf{05bc9eca84c44570670df15d29a37d78ffe9813073ca41d2147b14a06e9d96a1}
```

## 方法总结

本题的核心不是暴力枚举损坏的 Base64，而是先用 OpenSSH 二进制格式识别仍然完整的 RSA 数学字段，再把 `iqmp` 转换成模 $n$ 的多项式约束。$q$ 的已知高、中、低片段让剩余未知量满足小根条件，二元 Coppersmith 因而能够恢复素因子。最后一步还必须区分“数学上等价的私钥”和“字节级相同的原文件”：随机 check integer、换行和序列化格式都会改变题目要求的 SHA-256。
