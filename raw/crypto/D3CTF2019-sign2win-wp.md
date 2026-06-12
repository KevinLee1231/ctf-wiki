# sign2win

## 题目简述

题目提供 ECDSA 签名与验证服务，最终要求提交一个签名，使它能同时作为两个不同消息的合法签名通过验证。利用的是 ECDSA 验证公式的可构造性：选定相同的 nonce `k` 使两个签名拥有相同 `r`，再构造特殊私钥，使同一个 `(r,s)` 对两个不同摘要都满足验证等式。

题目资料地址：https://github.com/BubbLess/d3ctf_sign2win

`server.py` 先做 4 字节前缀 PoW，菜单包含生成密钥、提交公钥、签名消息、验证消息和获取 flag。关键是选项 2 允许用户用 `gvk.from_string(msg, curve=SECP256k1)` 提交自己的验证公钥；选项 5 固定要求消息 `I want the flag` 与 `I hate the flag` 的两个签名完全相同，并且都能在当前 `vk` 下验证通过。这个服务端条件正是构造特殊公钥/私钥的入口。

## 解题过程

题目给了一个ECDSA的签名与验证的服务，观察最后获取flag的要求，需要提供一个签名，可以使用两个不同的消息来验证它并通过，这里其实是ECDSA的一个特性，可以寻找到一个私钥来满足对不同消息的重复的签名

假设我们需要签名的消息为m1和m2，则它们的摘要分别为h1=H(m1)，h2=H(m2)，然后我们选择一个随机数k来计算签名中的r，因为要生成相同的签名那也就是r和s都相同，首先r相同那就表示签名所选择的随机数k是相同的，这样我们就有

$(x_1, y_1) = k \cdot G$

$r = x_1 \bmod n$

这样利用r,h1,h2我们就可以计算对h1的签名在h2也能验证通过的私钥了

$\text{pri} = -\frac{h_1 + h_2}{2r} \bmod n$

下面我们不妨简单验证下，用pk签名我们可以得到s

$s = k^{-1}(h + \text{pri} \cdot r) \bmod n$

验证签名要求下面的等式成立

$r = h \cdot s^{-1} \cdot G + r \cdot s^{-1} \cdot \text{pub} \bmod n$

$r = h \cdot s^{-1} \cdot G + r \cdot s^{-1} \cdot \text{pri} \cdot G \bmod n$

$r = s^{-1} \cdot G \cdot (h + r \cdot \text{pri}) \bmod n$

从前面对pri的计算我们可以得到

$r \cdot \text{pri} = -\frac{h_1 + h_2}{2} \bmod n$

这样假设我们签名的消息是h1时，前面的等式就如下

$r = s^{-1} \cdot G \cdot \left(h_1 - \frac{h_1 + h_2}{2}\right) \bmod n$

$r = G \cdot \frac{h_2 - h_1}{2s} \bmod n$

同样的将pri代入s，即有

$s = \frac{h_2 - h_1}{2k} \bmod n$

代回前面的等式，就能得到

$r = G \cdot k \bmod n$

等式成立，同理可得签名的消息为h2时等式依然成立

exp如下

```python
import ecdsa
import gmpy
from ecdsa import SECP256k1
import hashlib

m1="I want the flag"
m2="I hate the flag"

ks = 70072565845091379839538401416782237438929290760763328213667318793346806056450
r=23372277234339732161528747619365498567249265222314495344099167639942101343337

sk = ecdsa.SigningKey.generate(curve=SECP256k1)

n = sk.curve.order

h1=hashlib.sha256(m1.encode("utf-8"))
hs=h1.digest()
z1 = ecdsa.util.string_to_number(h1.digest())%n
h2=hashlib.sha256(m2.encode("utf-8"))
z2 = ecdsa.util.string_to_number(h2.digest())%n

x=-((z1+z2)* gmpy.invert(2*r, n)) %n

sk1 = sk.from_secret_exponent(x,sk.curve)

vk1=sk1.get_verifying_key()
print('pubkey:',vk1.to_string().hex())

sig=sk1.sign(m1.encode('utf-8'),k=ks,hashfunc=hashlib.sha256)
print('sign:',sig.hex())
```

## 方法总结

- 核心技巧：ECDSA 验证等式允许通过选择特殊私钥构造“同一签名验证两个消息”的重复签名条件。
- 识别信号：服务要求一个签名同时验证两个不同消息，且允许提交公钥或影响私钥生成时，应检查 ECDSA 验证公式是否可被代数构造。
- 复用要点：构造式依赖 $2r$ 在曲线阶模数下可逆；实现时要明确消息哈希到整数的方式、曲线阶 `n`、nonce `k` 和签名编码格式。

