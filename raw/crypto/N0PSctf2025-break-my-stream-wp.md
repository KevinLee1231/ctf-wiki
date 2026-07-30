# Break My Stream

## 题目简述

服务实现了一种自定义流密码：启动时随机生成一次 256 字节密钥，先输出加密后的 flag，随后允许用户反复提交明文并获得对应密文。目标是利用实现缺陷恢复 flag。

## 解题过程

流密码的加密形式可以写成：

$$
C=P\oplus K_s
$$

其中 $K_s$ 是伪随机算法产生的密钥流。只要同一段密钥流被重复用于两个消息，就可以通过一组已知明文与密文恢复密钥流。

源码虽然只生成一次随机密钥，但每次加密都重新构造对象：

```python
ct = CrypTopiaSC(key).encrypt(pt)
```

`CrypTopiaSC` 的构造函数会从同一密钥重新执行 KSA，并让 PRGA 从初始状态开始。因此，flag 与用户输入的第 $i$ 个字节始终使用相同的第 $i$ 个密钥流字节。提交与 flag 等长的全零明文后，有：

$$
C_0=0\oplus K_s=K_s
$$

再将服务最初给出的 `enc_flag` 与 $C_0$ 异或即可恢复明文。完整利用脚本如下：

```python
import sys
from pwn import remote, xor

io = remote(sys.argv[1], int(sys.argv[2]))

io.recvuntil(b"thing: ")
enc_flag = bytes.fromhex(io.recvline().strip().decode())

known_plaintext = b"\x00" * len(enc_flag)
io.recvuntil(b"message: ")
io.sendline(known_plaintext)
known_ciphertext = bytes.fromhex(io.recvline().strip().decode())

keystream = xor(known_plaintext, known_ciphertext)
print(xor(enc_flag, keystream).decode())
```

运行后得到：

```text
N0PS{u5u4L_M1sT4k3S...}
```

## 方法总结

问题不在自定义 KSA 或 PRGA 的具体强度，而在状态生命周期：程序为每次请求重新初始化流密码，导致相同密钥流反复从头使用。审计流密码时，应优先检查 nonce、计数器和内部状态是否会复用；一旦存在密钥流复用，选择全零明文通常能最直接地把密钥流本身取出来。
