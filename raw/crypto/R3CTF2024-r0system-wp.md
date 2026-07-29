# r0system

## 题目简述

服务实现了注册、密码或令牌登录、重置密码、查看 ECDH 密钥以及公共频道等功能。Alice 与 Bob 在公共频道公布公钥，并用双方的 ECDH 共享密钥加密 flag。目标不是破解椭圆曲线，而是从账户系统中取得任意一方的私钥。

附件中的 `reset_password` 只检查目标用户名是否存在，没有验证调用者是否就是该用户：

```python
def reset_password(self, username, new_password):
    if username not in self.usernames:
        return False, b"Username does not exist!"
    self.passwords[username] = new_password
    return True, b"Reset password successfully!"
```

## 解题过程

先注册一个普通账号并登录。进入登录后的服务菜单，调用重置密码功能，把目标用户名指定为 `AliceIsSomeBody`，新密码设为自选值。由于函数没有所有权检查，Alice 的密码会被直接覆盖。

退出当前账号后，使用新密码登录 Alice。服务允许已登录用户查看自己的 ECDH 私钥，因此可以取得 Alice 的私钥。同时从公共频道记录中读取：

- Alice 的公钥；
- Bob 的公钥；
- Alice 发给 Bob 的加密 flag。

附件 `utils.py` 表明共享密钥的派生方式不是标准 KDF，而是对椭圆曲线共享点的字符串表示取 MD5：

```python
def exchange_key(self, others_publickey):
    point = self.curve.mul(self.private_key, others_publickey)
    return md5(str(point).encode()).digest()
```

因此使用 Alice 私钥乘 Bob 公钥即可重算共享点，再按照完全相同的序列化和 MD5 过程得到 16 字节 AES 密钥：

```python
shared_point = curve.mul(alice_private_key, bob_public_key)
key = md5(str(shared_point).encode()).digest()
plaintext = AES.new(key, AES.MODE_ECB).decrypt(encrypted_flag)
```

最后按附件的填充规则去除尾部填充，得到本实例 flag。这里不需要离散对数攻击；账户越权已经直接泄露了 ECDH 的长期私钥。

完整的交互实现和同系列密码分析可参考 [R3CTF 2024 Crypto Writeup](https://tang.cat/2024/06/10/R3CTF-2024-Crypto-Writeup.html)。正文已经包含本题所需的越权点、密钥派生方式和解密链，链接主要用于核对原始交互细节。

## 方法总结

本题的决定性缺陷是密码重置接口缺少目标账户授权，而不是 ECDH 或 AES 本身。遇到“加密消息 + 账户系统”的组合题时，应先审计身份边界：重置密码、令牌登录和密钥查看只要有一处越权，就会把密码问题降级成普通解密。恢复共享密钥时还必须严格复现附件中的点序列化与 MD5 过程，否则即使椭圆曲线运算正确也无法得到相同 AES 密钥。
