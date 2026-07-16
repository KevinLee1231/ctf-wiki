# week1simpleBabyEasyRSA

## 题目简述

题目直接给出 RSA 的两个素数 $p,q$、公钥指数 $e$ 和密文 $c$。恢复私钥 $d$ 与明文整数 $m$ 后，需要拼接十进制字符串 `d || m`，计算其 32 位小写 MD5，并用 `0xGame{}` 包裹。

## 解题过程

已知：

~~~text
p = 59
q = 97
e = 37
c = 3738
~~~

由 $n=pq=5723$，计算：

$\varphi=(p-1)(q-1)=5568$

$d=\operatorname{inverse}(e,\varphi)=301$

$m\equiv c^d\pmod n=5499$

题目要求拼接的是十进制文本 `3015499`，不是整数相加，也不包含分隔符：

```python
from hashlib import md5

d, m = 301, 5499
digest = md5(f"{d}{m}".encode()).hexdigest()
print(f"0xGame{{{digest}}}")
```

得到：

```text
0xGame{d42b66fc047f9e80922f1a2b11e589c0}
```

## 方法总结

- 核心方法：利用已知 $p,q$ 计算 $\varphi(n)$ 和私钥指数，再完成 textbook RSA 解密及题目指定的 MD5 封装。
- 识别特征：RSA 全部私钥生成参数已知，难点不在分解，而在准确执行后处理规则。
- 注意事项：区分“连接”与数值相加；MD5 输入应是无分隔符的 ASCII 十进制字符串，输出需为 32 位小写十六进制。
