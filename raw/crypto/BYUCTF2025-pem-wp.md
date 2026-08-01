# PEM

## 题目简述

附件是一把可正常导入的 RSA/SSH 公钥。生成器并未从素数构造模数，而是先把一段包含 flag 的长字节串解释为整数 $x$，再设置 $n=x^2$，公钥指数仍为 $65537$。

因此 PEM 只是序列化外壳，真正的漏洞是模数被构造成完全平方数。

## 解题过程

用 RSA 库导入公钥并读取模数 $n$，对其求精确整数平方根。根就是原始字节串对应的整数，无需分解 RSA，也无需私钥：

```python
from Crypto.PublicKey import RSA
from Crypto.Util.number import long_to_bytes
from math import isqrt

key = RSA.import_key(open("ssh_host_rsa_key.pub", "rb").read())
x = isqrt(key.n)
assert x * x == key.n
print(long_to_bytes(x).decode())
```

输出的前后是随机填充文本，中间可直接读到：

```text
byuctf{P3M_f0rm4t_1s_k1ng}
```

## 方法总结

- 核心技巧：解析公钥参数后识别完全平方模数，直接整数开根恢复编码消息。
- 识别信号：RSA 公钥文件本身并不保证参数安全；题目若只给 PEM/SSH key，仍应检查 $n$ 的位数、平方性、小因子和其他特殊结构。
- 复用要点：使用整数平方根并验证 $x^2=n$，不要用浮点 `sqrt`；恢复整数后按大端字节转换。
