# L3akCTF 2025 Breadcrumbs Writeup

## 题目简述

题目提供三个互相补充的证据源：

| 工件 | 文件名 | 主要用途 |
| --- | --- | --- |
| 内存镜像 | `memdump.mem` | 找到入口进程、旁加载 DLL、注入载荷和用户凭据 |
| 网络流量 | `traffic.pcapng` | 解密 TLS 并导出攻击者下发的文件 |
| 磁盘镜像 | `disk.ad1` | 恢复 DPAPI master key、持久化文件和加密数据 |

flag 也分散在三层中。必须沿着“内存定位恶意链 → 网络恢复下发内容 → 磁盘与内存联合解 DPAPI”的顺序，把 breadcrumbs 串成一条完整事件链。

## 解题过程

### 内存：从异常进程树找到旁加载 DLL

先确认镜像环境：

```bash
python3 vol.py -f memdump.mem windows.info
python3 vol.py -f memdump.mem windows.pslist
```

镜像来自 64 位 Windows 10 build 19041，系统时间为 `2025-06-16 13:39:47 UTC`。进程树中有一组异常关系：

| PID | PPID | 进程 |
| --- | --- | --- |
| 3240 | 4900 | `7za.exe` |
| 5460 | 3240 | `conhost.exe` |
| 1472 | 3240 | `msedge.exe` |

压缩工具不应同时拉起控制台宿主和 Edge。通过 `windows.filescan` 找到：

```text
\Users\abdelrhman322\Downloads\x64\7za.exe
\Users\abdelrhman322\Downloads\x64\cryptbase.dll
```

从内存导出 `7za.exe` 后应先去掉末尾因节映射而补入的 `00`，再计算哈希。其 SHA-256 为

```text
4b2b76f03d586d216ebe019ca118dc082b7fe6cce0af676d297281f557edd00e
```

这是正常的 7-Zip 程序。异常点是同目录的 `cryptbase.dll`：系统 DLL 正常位置应在 `C:\Windows\System32`，把同名 DLL 放在可信 EXE 旁边正是 DLL side-loading。

```bash
python3 vol.py -f memdump.mem windows.dumpfiles \
  --virtaddr 0xd484d8922bc0
```

### 逆向 `cryptbase.dll` 和内嵌载荷

导出的 DLL 会：

1. 以 `CREATE_SUSPENDED` 创建 `msedge.exe`；
2. 从自身资源中读取类型 `SHELL`、ID `0x65` 的数据；
3. 用 16 字节循环密钥 `X7qP9zLmA2VtEjC0` 异或解密；
4. `VirtualAllocEx`、`WriteProcessMemory`、`CreateRemoteThread`，在 Edge 中运行载荷。

解密资源：

```python
key = b"X7qP9zLmA2VtEjC0"
enc = open("SHELL.bin", "rb").read()
plain = bytes(x ^ key[i % len(key)] for i, x in enumerate(enc))
open("decrypted_shellcode.bin", "wb").write(plain)
```

载荷开头的 `call $+5; pop` 等位置无关代码模式表明它是 ShellcodeRDI 封装的 DLL。搜索 `MZ` 可在偏移 `0xB28` 找到原 PE，也可用 `binwalk` carving。还原的下载器访问 `10.10.70.114`，下载：

```text
encrypted.bin
2.txt
sctasks.exe
```

它还请求：

```text
https://10.10.70.114/L3AK{AV_evasion_is_easy
```

因此第一段为：

```text
L3AK{AV_evasion_is_easy
```

### 网络：利用弱 RSA 证书解 TLS

在 PCAP 中过滤：

```text
ip.addr == 10.10.70.114
```

受害者 `10.10.70.140` 与 C2 使用 TLS 1.2，套件为：

```text
TLS_RSA_WITH_AES_256_GCM_SHA384
```

该套件使用静态 RSA 密钥交换，没有前向保密。抓包中的服务器证书公钥只有 1323 位，而且模数由两个距离极近的素数组成。把证书 DER 导出后：

```bash
openssl x509 -in certificate.crt -inform DER -text -noout -modulus
```

对模数 $n=pq$ 可直接用 Fermat 分解。令 $a=\lceil\sqrt n\rceil$，若

$$
b^2=a^2-n
$$

为完全平方数，则

$$
p=a-b,\qquad q=a+b.
$$

本题两个因子只相差 360，几乎立刻命中：

```python
from math import isqrt

a = isqrt(n)
if a * a < n:
    a += 1
while True:
    b2 = a * a - n
    b = isqrt(b2)
    if b * b == b2:
        p, q = a - b, a + b
        break
    a += 1
```

用 $p,q,e=65537$ 重建 PEM 私钥，在 Wireshark 的 TLS RSA Keys List 中配置：

```text
IP: 10.10.70.114
Port: 443
Key file: privatekey.pem
```

重新加载抓包后可导出三个 HTTP 对象。其中 `2.txt` 给出第二段：

```text
_Mastering_forensics_
```

### 磁盘：从 PyInstaller stealer 找到 DPAPI 用法

`sctasks.exe` 含大量 Python 符号，是 PyInstaller 产物。用 `pyinstxtractor` 解包后，入口之一为 Python 3.13 的 `browser_stealer.pyc`。反编译代码显示它会：

- 提取 Chromium 密码、cookie、信用卡、历史和下载记录；
- 用浏览器 `Local State` 中的主密钥和 Windows DPAPI 解密凭据；
- 把结果以 JSON POST 到 `http://10.10.70.114:443`；
- 调用 `CryptUnprotectData` 解开 `encrypted.bin`，取得另一层 AES 密钥。

关键逻辑为：

```python
protected_key = open("encrypted.bin", "rb").read()
aes_key_hex = CryptUnprotectData(
    protected_key, None, None, None, 0
)[1]
```

DPAPI blob 与用户、SID 和 master key 绑定，不能脱离系统上下文直接解。磁盘镜像中对应目录为：

```text
Users\abdelrhman322\AppData\Roaming\Microsoft\Protect\
└── S-1-5-21-2532670039-4151104164-2696135040-1001
    ├── 4dc3472c-8370-4831-9124-f45a6d742757
    └── Preferred
```

内存中的 `windows.hashdump` 给出用户 `abdelrhman322` 的 NTLM：

```text
7ed4bd1015f33ad80eff4a63119ef2d9
```

题目字典可将它恢复为密码：

```text
5563756
```

据此解 master key，再解 DPAPI blob：

```text
dpapi::masterkey /in:"4dc3472c-8370-4831-9124-f45a6d742757" /sid:S-1-5-21-2532670039-4151104164-2696135040-1001 /password:5563756

dpapi::blob /in:"encrypted.bin" /masterkey:<上一步输出的 masterkey>
```

blob 中的明文为十六进制：

```text
6d795f73757065725f7365637265745f
```

即 AES key：

```text
my_super_secret_
```

### 解出第三段并拼接

stealer 中同时硬编码了 IV 和密文：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = bytes.fromhex("6d795f73757065725f7365637265745f")
iv = b"1234567891011123"
ct = bytes.fromhex(
    "6b4781995cf5e4e02c2625b3d1ac6389"
    "dbaf68fb5649a3c24ede19465f470412"
)

print(unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16))
```

得到：

```text
is_where_the_challenge_begins}
```

三段连接后为：

```text
L3AK{AV_evasion_is_easy_Mastering_forensics_is_where_the_challenge_begins}
```

仓库 README 指向的[出题人完整题解](https://abdelrahme.github.io/posts/L3akCTF-2025/)是本题缺失本地附件说明的主要来源。外链中与求解有关的进程、密钥、DPAPI 路径和密文已经在本文展开。

## 方法总结

这题考查的是跨证据源关联，而不是某一个工具。内存说明恶意代码如何进入 Edge，并给出 C2 与第一段；PCAP 中的弱 RSA 证书允许还原 TLS 文件和第二段；磁盘中的 DPAPI master key 再与内存凭据结合，解出第三段。每一步都应保留“工件来源 → 恢复动作 → 新线索”的映射，否则很容易在长攻击链中失去方向。
