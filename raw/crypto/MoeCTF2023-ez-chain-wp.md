# ez_chain

## 题目简述

题目先把 72 字节 flag 解释成大整数，再按 $B=\operatorname{bytes\_to\_long}(\texttt{b"koito"})$ 展开为高位在前的 $B$ 进制数字 $M_i$。随后使用固定初始值 $IV=3735927943$ 和同一个整数密钥 $K$ 做异或链：

$$
C_i=M_i\oplus C_{i-1}\oplus K,\qquad C_{-1}=IV.
$$

## 解题过程

这不是标准 CBC：没有分组密码，链中只有异或。只要知道任意一组相邻的 $M_i,C_{i-1},C_i$，就能直接恢复

$$
K=M_i\oplus C_{i-1}\oplus C_i.
$$

flag 的总长度固定为 72 字节且前缀固定为 `moectf{`。最高位的 $B$ 进制数字只由消息开头决定，因此可以用同长度的占位 flag 计算正确的 $M_0$；后面的未知字节不会进入这个最高位数字。由 $M_0$、$IV$ 和 $C_0$ 即可得到密钥，再逐项解链。

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes

base = bytes_to_long(b"koito")
iv = 3735927943
cipher = [
    8490961288, 122685644196, 349851982069, 319462619019,
    74697733110, 43107579733, 465430019828, 178715374673,
    425695308534, 164022852989, 435966065649, 222907886694,
    420391941825, 173833246025, 329708930734,
]

def blockize(value):
    digits = []
    while value:
        digits.append(value % base)
        value //= base
    return digits[::-1]

def deblockize(digits):
    value = 0
    for digit in digits:
        value = value * base + digit
    return value

known_shape = b"moectf{" + b"A" * 64 + b"}"
assert len(known_shape) == 72
m0 = blockize(bytes_to_long(known_shape))[0]
key = m0 ^ iv ^ cipher[0]

plain_digits = []
previous = iv
for current in cipher:
    plain_digits.append(current ^ previous ^ key)
    previous = current

flag = long_to_bytes(deblockize(plain_digits))
print(flag)
```

输出为：

```text
moectf{thE_c6c_Is_not_so_hard_9ifxi9i!JGofMJ36D9cPMxroif6!M6oSMuliPPcA3}
```

## 方法总结

看到“链式加密”不能直接套用 CBC 结论，应先把递推式写出来。本题的重复密钥与纯异或结构使一次已知明文泄露整个链；固定消息长度又保证最高位基数数字可以仅由已知前缀确定。
