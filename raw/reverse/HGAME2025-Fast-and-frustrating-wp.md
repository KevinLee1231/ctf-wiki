# Fast and frustrating

## 题目简述

附件是一个经过 .NET NativeAOT 编译的 x64 程序，并附有调试符号。程序先检查当前 UI 区域代码是否为 `vt`，再从区域资源中读取压缩约束、加密 flag 和 HKDF 的 `info`。正确输入必须满足一个整数线性方程组，随后又被用作 HKDF 的输入密钥材料，派生 AES-256-CBC 的密钥与 IV。

题目作者公开的[归档源码仓库](https://github.com/CSharperMantle/hgame2025_FastAndFrustrating_public)包含 `Program.cs` 和 `Resources.vt.resx`，可用于核对从 NativeAOT 可执行文件中恢复出的程序逻辑和资源值；以下正文已写出解题所需的约束形式、关键资源和解密参数，不依赖阅读外链才能理解。

## 解题过程

程序入口首先检查：

```csharp
if (CultureInfo.CurrentUICulture.TwoLetterISOLanguageName != "vt") {
    Console.WriteLine("No way! You must be a Vidar-Team member to run this app.");
    return;
}
```

直接修改分支或静态提取 `vt` 卫星资源即可绕过这层干扰。随后程序读取输入并加载 `Constrs`：

```csharp
var password = Encoding.ASCII.GetBytes(Console.ReadLine() ?? "");

using var stream = new GZipStream(
    new MemoryStream(Convert.FromBase64String(Constrs)),
    CompressionMode.Decompress
);

var constrs = JsonSerializer.Deserialize(
    stream,
    ConstrsJsonContext.Default.Constrs
);
```

解压后的 JSON 有两个字段：二维整数数组 `mat_a` 和整数数组 `vec_b`。后续 LINQ 逻辑等价于：

```csharp
var match = constrs.MatA
    .Zip(constrs.VecB)
    .All(t => t.First
        .Zip(password.Take(t.First.Length))
        .Select(p => p.First * p.Second)
        .Sum() == t.Second);
```

把 `mat_a` 记作矩阵 $A$，`vec_b` 记作向量 $b$，输入的 ASCII 字节记作 $x$，校验条件就是：

$$
\forall i,\quad \sum_j A_{i,j}x_j=b_i,
$$

即：

$$
Ax=b.
$$

在 .NET NativeAOT 可执行文件中，资源数据位于 `.rdata`。`.resources` 文件头的魔数是 `0xBEEFCACE`，所以在小端文件中搜索字节 `CE CA EF BE`，即可定位资源并提取 `Constrs`、`Flag`、`KeyInfo`。其中：

```text
Flag    = GFxmVucV6MVUXiWCMAnWpyvzXoLdHc5CmFeim+JjUBszB8HFX8Ku8NMc201AGZ9X
KeyInfo = HGAME2025
```

将提取出的 `Constrs` Base64 字符串保存到 `constrs_b64.txt`，下面的代码可解压 JSON 并精确求解整数方程组：

```python
import base64
import gzip
import json

from sympy import Matrix

constrs_b64 = open("constrs_b64.txt", "r", encoding="ascii").read().strip()
constraints = json.loads(gzip.decompress(base64.b64decode(constrs_b64)))

matrix_a = Matrix(constraints["mat_a"])
vector_b = Matrix(constraints["vec_b"])
solution = matrix_a.inv() * vector_b

if any(value.q != 1 for value in solution):
    raise ValueError("方程组没有整数解")

password = bytes(int(value) for value in solution)
print(tuple(password))
print(password.decode("ascii"))
```

求得的字节向量为：

```text
(67, 111, 109, 112, 114, 101, 115, 115, 101,
 100, 69, 109, 98, 101, 100, 100, 101, 100,
 82, 101, 115, 111, 117, 114, 99, 101, 115)
```

转换为 ASCII：

```text
CompressedEmbeddedResources
```

程序随后使用 SHA-256 HKDF，参数如下：

- 输入密钥材料 `IKM`：`CompressedEmbeddedResources`
- `salt`：空
- `info`：`HGAME2025`
- 输出长度：48 字节

因为 .NET `Aes.Create()` 的默认密钥长度是 256 位、分组长度是 128 位，所以派生结果的前 32 字节是 AES 密钥，后 16 字节是 CBC IV：

```text
key = d924da2c855de9a1966505f98cd80c022275b49788158f12de26bf5d8c01f9e7
iv  = 3b55deb78efa547db1fa9f2f5ffb592d
```

完整解密代码如下：

```python
import base64

from Crypto.Cipher import AES
from Crypto.Hash import SHA256
from Crypto.Protocol.KDF import HKDF

password = b"CompressedEmbeddedResources"
key_info = b"HGAME2025"
ciphertext = base64.b64decode(
    "GFxmVucV6MVUXiWCMAnWpyvzXoLdHc5CmFeim+JjUBszB8HFX8Ku8NMc201AGZ9X"
)

derived = HKDF(
    master=password,
    key_len=48,
    salt=b"",
    hashmod=SHA256,
    context=key_info,
)

key = derived[:32]
iv = derived[32:]
plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)

padding_length = plaintext[-1]
if plaintext[-padding_length:] != bytes([padding_length]) * padding_length:
    raise ValueError("PKCS#7 padding 无效")

print(plaintext[:-padding_length].decode("utf-8"))
```

输出 flag：

```text
hgame{F4st_4nd_frustr4t1ng_A0T_compilat1on}
```

## 方法总结

本题把核心数据藏在特定区域的 `.resources` 中，并用 NativeAOT 增加静态分析难度。完整链路是：识别 `vt` 区域资源，按 `CE CA EF BE` 定位资源文件，Base64 解码并 GZip 解压 `Constrs`，将 LINQ 校验还原成整数方程组 $Ax=b$，求出输入 `CompressedEmbeddedResources`，最后按 SHA-256 HKDF 派生 32 字节密钥和 16 字节 IV，执行 AES-256-CBC/PKCS#7 解密。CyberChef 可以复核单个步骤，但脚本更能明确字节边界、HKDF 参数和填充方式。
