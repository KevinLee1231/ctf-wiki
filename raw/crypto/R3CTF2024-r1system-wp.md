# r1system

## 题目简述

r1system 延续 r0system 的账户与加密通信模型：Alice 和 Bob 通过 ECDH 协商共享密钥，公共频道给出双方公钥和 AES-ECB 加密的 flag。r0system 的任意密码重置已经修复，但注册用户名的保留账号检查写错了变量。

服务本来应同时禁止注册 Alice 和 Bob，实际条件却重复判断了 Alice：

```python
if username == AliceUsername or username == AliceUsername:
    return False, b"Username is not allowed!"
```

因此 Alice 仍受保护，Bob 却可以被普通用户重新注册。

## 解题过程

直接把注册用户名设为 `BobCanBeAnyBody`，密码使用任意自选值。错误的条件无法命中 Bob，后续注册逻辑会为这个用户名写入新的密码、令牌和 ECDH 密钥材料。

随后使用自选密码登录 Bob，并通过“查看密钥”功能取得 Bob 的私钥。公共频道中同时给出了 Alice 公钥、Bob 公钥以及密文，因此按附件实现恢复共享密钥：

```python
shared_point = curve.mul(bob_private_key, alice_public_key)
key = md5(str(shared_point).encode()).digest()
flag_padded = AES.new(key, AES.MODE_ECB).decrypt(encrypted_flag)
```

ECDH 的交换律保证：

$$
d_BQ_A=d_B(d_AG)=d_A(d_BG)=d_AQ_B.
$$

所以取得 Bob 私钥与取得 Alice 私钥等价。最后按照服务的自定义填充格式去除末尾字节即可得到实例 flag。

同系列的完整交互脚本可参考 [R3CTF 2024 Crypto Writeup](https://tang.cat/2024/06/10/R3CTF-2024-Crypto-Writeup.html)。关键漏洞、共享密钥推导与解密步骤已经在正文中给出，阅读外链不是复现本题的前提。

## 方法总结

漏洞来自保留用户名校验中的复制粘贴错误。审计这类条件时不能只看“存在黑名单判断”，而要逐个确认每个敏感身份都被实际比较。拿到任意通信方的 ECDH 私钥后，不需要攻击另一方或破解曲线；复现题目使用的点编码、MD5 和 AES 模式即可完成解密。
