# RSA

## 题目简述

服务把输入整数直接做原始 RSA 签名，返回 $s=m^d\bmod n$。公钥指数 $e$ 和模数 $n$ 同时公开；附件中的密文是干扰项，真正的 flag 就是被签名的消息。

## 解题过程

RSA 签名验证会计算

$$
s^e\bmod n=(m^d)^e\bmod n=m.
$$

因此无需分解 $n$，也无需攻击附件密文。向服务请求一次 flag 对应消息的签名后，把返回值按公钥指数幂模 $n$ 即可还原整数消息：

```python
m = pow(signature, e, n)
flag = m.to_bytes((m.bit_length() + 7) // 8, "big")
print(flag.decode())
```

结果为：

```text
n00bz{pl34s3_s1gn_h3r3_4nd_h3r3_4nd_h3r3...}
```

## 方法总结

原始 RSA 签名本质上是私钥幂运算，验证过程天然恢复被签名整数。实际协议必须先对消息做带域分离的安全编码和哈希，例如 RSA-PSS；绝不能把秘密消息本身直接交给可公开验证的签名接口。
