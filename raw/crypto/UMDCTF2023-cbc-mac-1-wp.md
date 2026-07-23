# CBC-MAC 1

## 题目简述

服务允许查询任意整块长度消息的 CBC-MAC，并要求提交一个从未查询过的消息及其合法标签。实现使用固定的全零 IV：

```python
def cbc_mac(msg, key):
    cipher = AES.new(key, AES.MODE_CBC, iv=b"\x00" * 16)
    return cipher.encrypt(msg)[-16:]
```

CBC-MAC 只有在消息长度固定，或长度被安全绑定到认证过程时才具备标准安全性。本题允许不同长度消息共用同一密钥，因而可以延长已认证消息并复用中间链值。

## 解题过程

查询单个全零分组

$$
M=0^{128}.
$$

由于初始链值也是零，返回标签为

$$
t=E_k(0^{128}).
$$

构造一个未查询过的两分组消息

$$
M'=0^{128}\Vert t,
$$

并仍提交标签 $t$。其 CBC 链值为

$$
\begin{aligned}
C_1&=E_k(0^{128}\oplus0^{128})=t,\\
C_2&=E_k(t\oplus t)=E_k(0^{128})=t.
\end{aligned}
$$

因此 $\operatorname{MAC}(M')=t$，而服务只按完整字节串检查 `M'` 是否出现在查询列表中，长度不同的伪造消息不会被判为重复：

```python
message = b"\x00" * 16
tag = mac_query(message)

forged_message = message + tag
submit(forged_message, tag)
```

验证成功后得到：

```text
UMDCTF{Th!s_M@C_Sch3M3_1s_0nly_S3cur3_f0r_f!xed_l3ngth_m3ss4g3s_78232813}
```

## 方法总结

- 核心技巧：把已知 CBC-MAC 标签作为下一分组，使异或后的输入归零，从而延长消息却保持相同标签。
- 识别信号：零 IV 的裸 CBC-MAC 接受可变长度消息，又没有前缀自由编码或独立长度域时，应优先测试长度扩展伪造。
- 复用要点：CBC-MAC 的安全条件与“CBC 使用了安全分组密码”不是一回事；协议必须固定消息长度，或采用 CMAC 等专门设计。
