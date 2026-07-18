# week3CryptoSignin3

## 题目简述

已知教科书 RSA 密文 $c=m^e\bmod n$，要求计算 $5m$ 的加密结果，并提交其十进制末 25 位；不足 25 位时需在左侧补零。

## 解题过程

RSA 在没有填充时具有乘法同态性：

$$
E(5m)=(5m)^e\equiv5^em^e\equiv5^ec\pmod n.
$$

因此不需要知道私钥，也不需要恢复 $m$。仓库文本中的 $n,e,c$ 很长，直接从附件解析可避免抄录错误：

```python
import re
from pathlib import Path

text = Path("CryptoSignin3.txt").read_text(encoding="utf-8")

def value(name):
    match = re.search(rf"\b{name}\s*=\s*(\d+)", text)
    if not match:
        raise ValueError(f"缺少参数 {name}")
    return int(match.group(1))

n = value("n")
e = value("e")
c = value("c")
encrypted_5m = pow(5, e, n) * c % n
answer = str(encrypted_5m)[-25:].zfill(25)
print(f"0xGame{{{answer}}}")
```

输出：

```text
0xGame{8489769636593649908538102}
```

## 方法总结

教科书 RSA 的确定性和乘法同态性会泄露代数关系。本题只需把原密文乘以 $5^e\bmod n$；末位截取必须在十进制表示上完成，并按题意处理前导零。
