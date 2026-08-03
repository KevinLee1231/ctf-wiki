# UIUCTF 2023 Group Project Writeup

## 题目简述

服务基于 Diffie-Hellman 共享秘密派生 AES 密钥。它公布 $g=2$、随机素数 $p$ 和 $A=g^a\bmod p$，允许选手提供指数 $k$，随后计算：

$$
B=g^b\bmod p,\quad B_k=B^k\bmod p,\quad S=B_k^a\bmod p.
$$

AES 密钥为 `MD5(long_to_bytes(S))`，flag 使用 AES-ECB 加密。代码试图屏蔽 $k=1$、$p-1$ 和 $(p-1)/2$，但既没有限制 $k$ 的范围，也没有在检测失败后终止执行。

## 解题过程

直接提交 $k=0$。任意非零群元素的零次幂均为 1，因此：

$$
B_k=B^0\equiv1\pmod p,
$$

$$
S=B_k^a\equiv1^a\equiv1\pmod p.
$$

这样 AES 密钥完全固定为：

```python
hashlib.md5(long_to_bytes(1)).digest()
```

无需恢复任何 Diffie-Hellman 私钥。收到十进制密文后，将其转为字节并用已知密钥解密、去除 PKCS#7 填充即可：

```python
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad
from pwn import remote

io = remote("group.chal.uiuc.tf", 1337)
io.sendlineafter(b"Choose k = ", b"0")
io.recvuntil(b"c = ")
ciphertext = long_to_bytes(int(io.recvline()))

key = hashlib.md5(long_to_bytes(1)).digest()
plaintext = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), 16)
print(plaintext.decode())
```

得到 flag：

```text
uiuctf{brut3f0rc3_a1n't_s0_b4d_aft3r_all!!11!!}
```

## 方法总结

本题的决定性缺陷是攻击者可选指数却缺少有效域校验。$k=0$ 会把共享秘密退化为常量 1，令后续 AES 密钥可预测。即使列出若干“危险值”，只打印警告而不立即返回也无法形成安全边界。正确做法是严格验证 $1<k<p-1$、拒绝不合规输入并终止当前会话，同时验证参与密钥交换的群元素具有期望阶。
