# week2_re2

## 题目简述

题目是 Go 编译的 RSA 逆向。程序用 `fmt.Fscanf` 读取输入，将输入字节按大端序转换为 `big.Int`，计算 `input^e mod n` 后与硬编码密文比较。`n` 是十进制字符串，`c` 是八进制字符串，指数为 `65537`。

核心障碍是从 Go 静态链接二进制里提取 `n` 和 `c`，并注意密文的 `SetString` base 是 8 而不是 10。

## 解题过程

### 关键观察

在 `main.main` 汇编中可见：

```text
n: SetString(ptr=0x4cf1ac, len=0x269, base=10)
e: SetInt64(0x10001)
c: SetString(ptr=0x4cf415, len=0x2ab, base=8)
```

程序将输入构造成大整数：

```text
acc = acc * 256 + ord(c)
```

随后 `big.Int.Exp(acc, e, n)` 并与密文比较，完全符合 RSA 加密。

### 求解步骤

从内存提取 `n` 和八进制 `c`。`n` 可由 FactorDB 分解，随后用私钥解密：

```python
from Crypto.Util.number import long_to_bytes

n = int(n_str, 10)
c = int(c_str, 8)
e = 65537

# p, q 来自 FactorDB / RsaCtfTool
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
print(long_to_bytes(pow(c, d, n)).decode())
```

得到：

```text
0xGame{524b8387-5c6a-4f60-8a01-58f5c00e7ac4}
```

## 方法总结

- 核心技巧：从 Go `math/big` 调用中恢复 RSA 参数，并按正确进制解析密文。
- 识别信号：Go 二进制里出现 `big.Int.SetString`、`SetInt64(65537)`、`Exp`、`Cmp` 时，应考虑 RSA 校验。
- 复用要点：`SetString` 的 base 参数非常关键；本题密文是八进制，误按十进制会导致解密结果完全错误。
