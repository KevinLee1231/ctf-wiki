# 蔷薇地狱

## 题目简述

附件包含一个 Windows 文件加解密程序、两个 `.enc` 文件和一份抓包。程序在默认模式下用 `CryptGenRandom` 生成 32 字节 key 与 16 字节 IV，对当前目录文件进行 AES 加密，再把两者分别做无填充 RSA 加密，以十进制字符串发送到 `127.0.0.1:8080`；使用 `-d <key-hex> <iv-hex>` 则进入解密模式。恢复文件的关键是从 PCAP 取得两条 RSA 密文，分解只有 260 位的模数，按程序的小端整数表示还原对称密钥。

## 解题过程

### 确认程序行为与网络数据

追踪 `2.pcapng` 中的 TCP 数据，只有两个数据包携带应用层载荷，且载荷都是纯十进制整数，发送顺序与程序中的 key、IV 生成顺序一致：

```text
key_enc = 940562093044671832164601596047006087538730817712616191941210346596350554236362
iv_enc  = 266745019496886498654275776906007459419265791303737517308624461698148640999669
```

逆向命令行分支可恢复出接口：参数总数为 4 时从命令行读入 key 和 IV；第一个参数为 `-d` 时进入解密模式，否则执行加密。默认加密路径的关键行为可概括为：

```text
CryptGenRandom(provider, 32, key)
CryptGenRandom(provider, 16, iv)
send(decimal(RSA(key)), 127.0.0.1:8080)
send(decimal(RSA(iv)), 127.0.0.1:8080)
encrypt_files_in_current_directory(key, iv)
```

PE 导入表中的 `CryptGenRandom`、`CryptImportKey`、`CryptSetKeyParam`、`CryptEncrypt`、`CryptDecrypt` 与 `connect`、`send` 也与这条链一致。

### 识别 RSA 封装与字节序

进入发送函数后，可以看到数据先按小端解释成大整数，再转成十进制字符串。RSA 参数直接硬编码在程序中：

```text
e = 65537
N = 1294787702630841806287693875819503767329411267154281759595683996937344294354199
```

动态取一组随机 key，可验证程序满足 $c = m^e \bmod N$。例如内存中的 32 字节：

```text
2D 03 73 A2 8B B2 D9 53 1A 02 A5 29 F9 74 F5 6E
7F 1E 96 87 09 6D 53 5F AB 3F 57 89 C3 A2 82 12
```

按小端转换得到：

```text
m = 8372444747481441114806627663982646110080680876922988722792806344547391177517
```

计算 `pow(m, 65537, N)` 与程序发送前的整数完全相同，由此确认这里是没有填充的 textbook RSA。由于 $N$ 只有 260 位，使用 YAFU 的 SIQS 可以直接分解：

```text
p = 1330137249390799512563582841814876337813
q = 973424135910675582062565817184635606523
```

### 确认本地文件加密模式

程序构造的 CryptoAPI 明文密钥 Blob 中，类型为 `PLAINTEXTKEYBLOB`，算法标识 `0x6610` 对应 AES-256，密钥长度为 `0x20`。随后程序调用：

```text
CryptSetKeyParam(key, KP_MODE, CRYPT_MODE_ECB)
CryptSetKeyParam(key, KP_IV, iv)
```

因此实际文件算法是 AES-256-ECB。程序虽然仍生成、外传并设置 IV，但 ECB 不使用 IV；命令行接口仍要求提供第三个参数只是实现上的冗余。用恢复出的 32 字节小端 key 做 AES-ECB 解密并移除块填充，所得文件与 `Int.exe -d` 的输出逐字节一致。

### 恢复 key、IV 与文件

下面的脚本计算 RSA 私钥，按小端恢复 CryptoAPI 使用的 32 字节 key，并直接解密题目中的文本文件：

```python
import sys
from pathlib import Path

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

sys.stdout.reconfigure(encoding="utf-8")

p = 1330137249390799512563582841814876337813
q = 973424135910675582062565817184635606523
n = 1294787702630841806287693875819503767329411267154281759595683996937344294354199
e = 65537

key_enc = 940562093044671832164601596047006087538730817712616191941210346596350554236362
iv_enc = 266745019496886498654275776906007459419265791303737517308624461698148640999669

assert p * q == n
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

key_integer = pow(key_enc, d, n)
iv_integer = pow(iv_enc, d, n)

# 程序把随机字节流按小端解释成 RSA 明文整数。
key = key_integer.to_bytes(32, "little")
iv = iv_integer.to_bytes(16, "little")

print(f"key argument = {key_integer:064x}")
print(f"iv argument  = {iv_integer:032x}")

ciphertext = Path("Spreng's Reverse Note.txt.enc").read_bytes()
plaintext = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), AES.block_size)

start = plaintext.index(b"0xGame{")
end = plaintext.index(b"}", start) + 1
print(plaintext[start:end].decode("utf-8"))
```

恢复出的命令行参数为：

```text
key = 404ea31f4109911fb47cee98924fb6d0ee8788af8a3e0704d94a83cda804abd1
iv  = 90357aaef89ef45a8ef1db14ddd161eb
```

也可以让原程序在附件副本中完成解密：

```powershell
./Int.exe -d "404ea31f4109911fb47cee98924fb6d0ee8788af8a3e0704d94a83cda804abd1" "90357aaef89ef45a8ef1db14ddd161eb"
```

文本文件中恢复出的 flag 为：

```text
0xGame{I_4m_3o_S1eepy_(´～`)zzZ_zzZ_zzZ_zzZ}
```

## 方法总结

本题把文件加密器逆向、流量取证和弱 RSA 串在一起。看到随机生成的 key/IV、`send` 外传和 PCAP 中的长十进制整数，应沿数据流确认“字节序 → 大整数 → 模幂 → 十进制编码”的完整序列，而不是直接把抓包内容当作对称密钥。还要以实际 API 参数为准判断对称模式：存在 IV 并不必然代表 CBC，本题明确设置的是 AES-256-ECB，IV 只是被多余地保留在协议和命令行接口中。
