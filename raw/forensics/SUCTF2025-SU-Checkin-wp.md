# SU_Checkin

## 题目简述

附件是一份 HTTP 抓包。攻击者利用下载接口的目录穿越读取了 Spring Boot 进程和应用文件，却没有直接读到 flag。解题需要从流量中恢复 Java 加密算法、密文和不完整口令，再复现 `PBEWithMD5AndDES` 的密钥派生并补全剩余字符。

题目虽以 Web 目录穿越为故事背景，但选手面对的是已经发生的 PCAP，决定性工作是恢复和关联既有流量证据，因此归入 Forensics。

## 解题过程

先在 Wireshark 中筛选成功响应：

```text
http.response.code == 200
```

追踪 `tcp.stream eq 50` 后，可以看到多次 `GET /download?filename=...`。攻击者先尝试读取 `/root/flag.txt`，响应只有 `nonono`；随后读取三个真正有用的目标。

读取 `/proc/self/cmdline` 得到：

```text
java -jar suctf-0.0.1-SNAPSHOT.jar --password=SePassWordLen23SUCT
```

读取 `/proc/self/cwd/BOOT-INF/classes/hint` 得到：

```text
algorithm=PBEWithMD5AndDES
```

读取 `/proc/self/cwd/BOOT-INF/classes/application.properties` 得到：

```text
OUTPUT=ElV+bGCnJYHVR8m23GLhprTGY0gHi/tNXBkGBtQusB/zs0uIHHoXMJoYd6oSOoKuFWmAHYrxkbg=
```

可见口令前缀 `SePassWordLen23SUCT` 只有 19 个字符，而名字中的 `Len23` 提示完整口令长 23。按 flag 语境先补出 `F`，只需枚举后三位；即使不作该猜测，枚举四位也能用明文格式验证。

Java 的这一 PBE 实现把 Base64 解码结果的前 8 字节作为 salt，剩余部分作为 DES-CBC 密文。口令与 salt 拼接后连续做 1000 次 MD5，摘要前 8 字节是 DES key，后 8 字节是 IV：

```python
import base64
import hashlib
import itertools
import string
from Crypto.Cipher import DES

CIPHERTEXT = (
    "ElV+bGCnJYHVR8m23GLhprTGY0gHi/tNXBkGBtQusB/"
    "zs0uIHHoXMJoYd6oSOoKuFWmAHYrxkbg="
)


def decrypt(password: bytes) -> bytes | None:
    raw = base64.b64decode(CIPHERTEXT)
    salt, encrypted = raw[:8], raw[8:]

    digest = password + salt
    for _ in range(1000):
        digest = hashlib.md5(digest).digest()

    plaintext = DES.new(
        digest[:8],
        DES.MODE_CBC,
        digest[8:16],
    ).decrypt(encrypted)

    padding = plaintext[-1]
    if not 1 <= padding <= 8 or plaintext[-padding:] != bytes([padding]) * padding:
        return None
    return plaintext[:-padding]


alphabet = string.ascii_letters + string.digits
prefix = b"SePassWordLen23SUCTF"

for suffix in itertools.product(alphabet, repeat=3):
    password = prefix + "".join(suffix).encode()
    plaintext = decrypt(password)
    if plaintext and plaintext.startswith(b"SUCTF{") and plaintext.endswith(b"}"):
        print(password.decode())
        print(plaintext.decode())
        break
```

结果为：

```text
password = SePassWordLen23SUCTF666
flag     = SUCTF{338dbe11-e9f6-4e46-b1e5-eca84fb6af3f}
```

## 方法总结

这道签到题的关键是不要停在目录穿越本身，而要把三份泄漏拼成完整解密参数：`cmdline` 给口令前缀，`hint` 给算法，`application.properties` 给密文。爆破时应同时检查 PKCS#5/PKCS#7 padding 和 `SUCTF{...}` 格式，不能把“偶然能解成 UTF-8”当成正确口令。
