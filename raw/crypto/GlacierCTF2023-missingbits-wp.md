# GlacierCTF2023 - MissingBits

## 题目简述

附件给出缺失若干 Base64 行的 RSA 私钥 PEM，以及用对应公钥执行裸 RSA 得到的 256 字节密文。PEM 尾部仍能解码出 PKCS#1 `RSAPrivateKey` 的多个 ASN.1 INTEGER，其中包含私钥指数 $d$、素因子 $p$ 和 $q$。

## 解题过程

去掉 PEM 头尾，对仍存在的 Base64 行逐行解码并拼成十六进制。DER INTEGER 的标签是 `0x02`，随后是长度；长格式如 `02 82 01 00` 表示后续整数长 `0x100` 字节，`02 81 81` 表示长 `0x81` 字节。沿完整区域解析可识别：

- `02 03 01 00 01`：公钥指数 $e=65537$；
- `02 82 01 00 ...`：256 字节私钥指数 $d$；
- 后续两个 `02 81 81 ...`：带符号补零字节的 $p$、$q$。

头部缺失使原始 $n$ 字段不可直接读取，但 $n=pq$ 可以重建。题目没有使用 OAEP 或 PKCS#1 v1.5 padding，所以直接执行模幂即可：

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes

ct = open("ciphertext_message", "rb").read()
n = p * q
m = pow(bytes_to_long(ct), d, n)
print(long_to_bytes(m))
```

结果为：

```text
gctf{7hi5_k3y_can_b3_r3c0ns7ruc7ed}
```

## 方法总结

破损 PEM 不等于私钥信息全部丢失。应先恢复 Base64/DER 结构，再判断剩余整数能否代数重建缺失字段；只要 $d,p,q$ 仍在，模数和解密能力就都还在。解析 DER 时必须正确处理长格式长度和正整数前导 `00`。
