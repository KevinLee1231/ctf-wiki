# EzLogin-I

## 题目简述

注册接口禁止用户名中出现小写字符串 `admin`，登录接口却只在解密后的用户名严格等于 `admin` 时返回 flag1。Cookie 使用 AES-CBC 加密但没有消息认证码，因此可以修改 IV，对第一块明文实施 CBC 位翻转，把允许注册的 `Admin` 改成 `admin`。

## 解题过程

注册时服务端序列化的 JSON 以如下内容开头：

```text
{"username": "Admin", ...}
```

第一块恰好是 16 字节：

```text
{"username": "Ad
```

其中 `A` 位于块内偏移 14。CBC 第一块解密满足：

$$
P_1=D_K(C_1)\oplus IV
$$

因此只要修改 $IV[14]$，就能控制 $P_1[14]$。ASCII 大写 `A` 为 `0x41`，小写 `a` 为 `0x61`，差值为：

$$
0x41\oplus0x61=0x20
$$

将 Cookie 解码后对 IV 的第 14 字节异或 `0x20`，其余字节保持不变即可：

```python
from base64 import b64decode, b64encode
from pwn import remote

HOST = "TARGET"
PORT = 10005

io = remote(HOST, PORT)

io.sendlineafter(b">", b"R")
io.sendlineafter(b">", b"Admin")
io.recvuntil(b"[+] cookie : ")
cookie = bytearray(b64decode(io.recvline().strip()))

# IV 占前 16 字节；把明文中的 A 翻转为 a。
cookie[14] ^= 0x20

io.sendlineafter(b">", b"L")
io.sendlineafter(b">", b64encode(cookie))
print(io.recvline().decode().strip())
io.close()
```

修改后的明文用户名为 `admin`，JSON 结构和 PKCS#7 填充均未被破坏，登录接口返回：

```text
0xGame{ad34acff-a813-4bc3-a44a-c270edf244b7}
```

## 方法总结

AES-CBC 只提供机密性，不提供完整性。攻击者虽然不知道密钥，仍可通过修改前一密文块或 IV，精确翻转下一明文块的对应位。加密身份 Cookie 时必须使用带认证的 AEAD 模式，或至少对 `IV || ciphertext` 计算并验证独立的 MAC。
