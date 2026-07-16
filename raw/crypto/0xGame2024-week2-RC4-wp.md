# RC4

## 题目简述

服务先用随机 8 字节密钥加密一段用户可控明文，随后把 RC4 状态重新初始化为同一密钥，再加密 flag。两次加密从相同密钥流起点开始，因此提交与 flag 等长的全零明文即可直接获得密钥流，并与 flag 密文异或恢复明文。

## 解题过程

RC4 流加密满足：

$$
C=P\oplus K
$$

若选择 $P=0$，则 $C=K$。服务端在加密 flag 前执行：

```python
keystream = RC4(KEY)
c_user = Encrypt(user_plaintext, keystream)

keystream = RC4(KEY)
c_flag = Encrypt(flag, keystream)
```

所以两个密文异或为：

$$
C_{zero}\oplus C_{flag}=K\oplus(P_{flag}\oplus K)=P_{flag}
$$

flag 长 44 字节，输入接口接收十六进制文本，因此发送 88 个字符 `0`。完整脚本如下：

```python
from hashlib import sha256
from itertools import product
from string import ascii_letters, digits

from pwn import remote


io = remote("HOST", PORT)

# 完成 4 字符 SHA-256 PoW
io.recvuntil(b"sha256(XXXX+")
suffix = io.recvuntil(b")", drop=True)
io.recvuntil(b"== ")
target = io.recvline().strip().decode()

alphabet = ascii_letters + digits
for chars in product(alphabet, repeat=4):
    prefix = "".join(chars).encode()
    if sha256(prefix + suffix).hexdigest() == target:
        io.sendlineafter(b"XXXX: ", prefix)
        break

# 44 个零字节的十六进制表示
io.sendlineafter(b">", b"00" * 44)

io.recvuntil(b"c = ")
keystream = bytes.fromhex(io.recvline().strip().decode())
io.recvuntil(b"c = ")
flag_ciphertext = bytes.fromhex(io.recvline().strip().decode())

flag = bytes(a ^ b for a, b in zip(keystream, flag_ciphertext))
print(flag.decode())
io.close()
```

输出为：

```text
0xGame{81682337-6731-91c7-d060-3efcdfe1ba5f}
```

## 方法总结

RC4 等流密码绝不能在相同密钥流位置重复使用。已知或可选择一段明文时，攻击者能够恢复对应密钥流，再解开另一段同位置密文；本题通过重新初始化 RC4 明确制造了这一条件。
