# 不可加密的异世界

## 题目简述

题目提供 AES、DES 及 ECB、CBC、OFB、CFB 四种组合，并要求构造“加密后仍等于明文”的输入。三问分别考查 CBC 可控 IV、逐分组指定密文，以及 CRC128 可逆性与 DES 弱密钥。

## 解题过程

### 疏忽的神：控制单分组 CBC

输入姓名后，明文为：

```python
your_pass = pad((your_name + "Open the door!").encode(), blocksize)
```

选择姓名 `1` 时，原字符串为 15 字节，PKCS#7 填充后恰好只有一个 AES 分组 $M_1$。CBC 第一块满足：

$$
C_1=E_K(M_1\oplus IV).
$$

希望 $C_1=M_1$，对任意自选 AES 密钥 $K$，只需令：

$$
IV=D_K(M_1)\oplus M_1.
$$

核心代码如下：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
from Crypto.Random import get_random_bytes

msg = pad(b"1Open the door!", 16)
key = get_random_bytes(16)
iv = bytes(a ^ b for a, b in zip(AES.new(key, AES.MODE_ECB).decrypt(msg), msg))

# 依次选择 AES、CBC，并把 key + iv 的十六进制提交给服务端
print((key + iv).hex())
```

服务端用该参数加密后，唯一密文块就是原明文块，从而通过第一问。

### 心软的神：逐轮指定目标密文块

第二问给出十个随机明文块 $M_1,\ldots,M_{10}$。第 $i$ 轮只检查第 $i$ 个输出块是否等于对应明文块，所以每轮可以重新选择 IV。

先指定目标 $C_i=M_i$。CBC 解密关系为：

$$
C_{j-1}=D_K(C_j)\oplus M_j.
$$

从 $j=i$ 开始向前递推，直到得到 $C_0$；在 CBC 中 $C_0$ 就是 IV。也就是说，只要已知密钥并能调用分组解密，就能把可控 IV 的自由度反向传播到任意一个密文块。

```python
def iv_for_target_block(key, blocks, pos):
    ecb = AES.new(key, AES.MODE_ECB)
    chain = blocks[pos]                 # 指定 C_pos = M_pos
    for j in range(pos, 0, -1):
        chain = bytes(a ^ b for a, b in zip(ecb.decrypt(chain), blocks[j]))
    return bytes(a ^ b for a, b in zip(ecb.decrypt(chain), blocks[0]))
```

对 `pos = 0..9` 分别计算 IV，并在每轮提交 `key + iv`。服务端虽然会加密全部十块，但每轮被检查的那一块都会精确等于原明文。

### 严苛的神：逆 CRC128 得到 DES 弱密钥

第三问不允许直接选密钥，而是令：

```python
your_key = long_to_bytes(crc128(your_pass), 16)[:blocksize]
```

选择 DES-ECB 后，只要让 CRC128 的高 8 字节成为 DES 弱密钥 `01 01 01 01 01 01 01 01` 即可。最方便的是令完整目标 CRC 为 16 个 `0x01`。

DES 弱密钥会让加密子密钥序列具有自反性，因此：

$$
E_K(E_K(M))=M.
$$

剩下的问题是构造 16 字节消息 $x$，使 `crc128(x)` 等于目标。固定输入长度时，CRC 是 $GF(2)$ 上的仿射映射。令：

$$
C=\operatorname{crc}(0^{128}),
$$

并对 128 个单位向量 $e_i$ 计算

$$
v_i=\operatorname{crc}(e_i)\oplus C.
$$

以 $v_i$ 为列组成 $128\times128$ 矩阵 $M$，则：

$$
\operatorname{crc}(x)=Mx+C.
$$

给定目标 $y$ 后，在 $GF(2)$ 上解线性方程：

$$
x=M^{-1}(y+C).
$$

用 SageMath 建矩阵并求解，得到 16 字节原像后验证：

```python
target = b"\x01" * 16
msg = crc_128_reverse(int.from_bytes(target, "big"))
assert len(msg) == 16
assert crc128(msg) == int.from_bytes(target, "big")
```

提交 `msg.hex()`，选择 DES 与 ECB。服务端从 CRC 取得弱密钥，对消息连续加密两次后仍得到原文，从而通过第三问。

## 方法总结

CBC 的 IV 不是装饰参数，它直接决定第一块，并可通过反向递推影响任意指定块；CRC 也不是密码学哈希，固定长度下可以用线性代数求原像；DES 弱密钥则提供自反加密。三问都没有攻破 AES 或暴力搜索密钥，而是利用工作模式、线性校验和弱密钥之间的结构性质，使“加密”退化为恒等变换。
