# Propagation

## 题目简述

服务把 user:{username},admin:false 加密成 AES-CBC token，并把 IV 与密文一并交给客户端。解密后只解析字段，没有 MAC 或 AEAD 标签。CBC 的前一密文块会异或到后一明文块；对于第一块，这个“前一块”就是可控 IV，因此可以精确改写第一块明文。

## 解题过程

CBC 第一块解密为：

$$
P_1=D_K(C_1)\oplus IV.
$$

若已知原明文块 $P_1$，希望改成 $P'_1$，只需发送：

$$
IV'=IV\oplus P_1\oplus P'_1.
$$

注册单字符用户名 a 后，第一块恰好是 user:a,admin:fal。把它改成 admin:true,user:，长度同为 16 字节：

~~~python
from Crypto.Util.strxor import strxor

token = register("a")
iv, ciphertext = token[:16], token[16:]
old = b"user:a,admin:fal"
new = b"admin:true,user:"
forged_iv = strxor(strxor(iv, old), new)
print(login(forged_iv + ciphertext))
~~~

后续明文虽然仍包含原字段的残余部分，但解析器先看到合法的 admin:true，权限判断成功，返回：

~~~text
maple{C1Ph3Rt3xt_t4mp3r1n}
~~~

## 方法总结

CBC 只保证在不知道密钥时难以恢复明文，不保证密文不可篡改。IV 若未被认证，第一块尤其容易被定向修改。安全 token 应使用 AES-GCM、ChaCha20-Poly1305 等 AEAD，或在加密后对 IV 与全部密文做 Encrypt-then-MAC，并拒绝任何标签校验失败的数据。
