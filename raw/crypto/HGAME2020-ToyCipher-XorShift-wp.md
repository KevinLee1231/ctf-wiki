# ToyCipher_XorShift

## 题目简述

自制分组密码把每个 8 字节块解释为 64 位整数，依次执行左移 13、右移 7、左移 17 的 xorshift，再用类似 CBC 的方式链接各块。xorshift 的每一步都是可逆线性变换，按相反顺序撤销即可解密。

## 解题过程

题目参数为：

```python
IV = b"c8C~M0d3"
ciphertext = bytes.fromhex(
    "15eb80358fe6f89b1802a5f3eb5a6ec6"
    "c33dc4f35822fb6e97e0b22be860a286"
    "02b35e2930a93ac5"
)
```

单块加密可写成：

$$
E(x)=L_{17}(R_7(L_{13}(x))),
$$

其中 $L_s(x)=x\oplus(x\ll s)$，$R_s(x)=x\oplus(x\gg s)$，所有结果截断到 64 位。反演左移异或时，低位会先稳定；反演右移异或时，高位会先稳定。迭代代回原方程直到 64 位范围全部确定即可：

```python
BLOCK_SIZE = 8
BITS = 64
MASK = (1 << BITS) - 1

def undo_left(value, shift):
    original = value
    for _ in range(BITS // shift + 1):
        original = value ^ ((original << shift) & MASK)
    return original & MASK

def undo_right(value, shift):
    original = value
    for _ in range(BITS // shift + 1):
        original = value ^ (original >> shift)
    return original & MASK

def decrypt_block(block):
    value = int.from_bytes(block, "big")
    value = undo_left(value, 17)
    value = undo_right(value, 7)
    value = undo_left(value, 13)
    return value.to_bytes(BLOCK_SIZE, "big")
```

链式加密满足 $C_i=E(P_i\oplus C_{i-1})$，其中 $C_{-1}=IV$，所以：

$$
P_i=E^{-1}(C_i)\oplus C_{i-1}.
$$

完整解密如下：

```python
def xor(left, right):
    return bytes(a ^ b for a, b in zip(left, right))

previous = IV
plain = b""

for offset in range(0, len(ciphertext), BLOCK_SIZE):
    block = ciphertext[offset:offset + BLOCK_SIZE]
    plain += xor(decrypt_block(block), previous)
    previous = block

pad_length = plain[-1]
assert plain.endswith(bytes([pad_length]) * pad_length)
plain = plain[:-pad_length]
print(plain.decode())
```

输出：

```text
hgame{tHi$+4lgOr1thM_i5_3@sY-t0~b2EaK}
```

题目源码与密文常量由 [HGAME2020 Crypto Writeup](https://blog.soreatu.com/posts/writeup-for-crypto-problems-in-hgame-2020/) 补齐；本文已把可逆原因、逆序和 CBC 关系完整写入正文。

## 方法总结

- 核心技巧：xorshift 不是单向函数，移位造成的三角依赖可从已稳定的一端逐步恢复。
- 关键顺序：先撤销最后的左移 17，再撤销右移 7，最后撤销左移 13；块外还要撤销 CBC 异或。
- 复用要点：模拟固定宽度整数时每步都要应用掩码；最后应严格验证填充，而不是无条件删除若干字节。
