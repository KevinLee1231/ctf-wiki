# mac-master

## 题目简述

服务端把消息认证码定义为 `MD5(message || KEY)`。虽然把密钥放在消息末尾避免了普通长度扩展攻击，但 MD5 的碰撞不抗附加相同后缀：若两个等长消息在处理完自身分组后进入同一内部状态，再追加同一个秘密密钥，最终摘要仍相同。

## 解题过程

官方脚本内置一对 64 字节 MD5 碰撞分组。两段消息只有少数字节不同，却满足：

$$
\operatorname{MD5}(M_1)=\operatorname{MD5}(M_2)
$$

并且碰撞发生在完整分组边界，所以对任意共同后缀 $S$ 都有：

$$
\operatorname{MD5}(M_1\Vert S)=\operatorname{MD5}(M_2\Vert S).
$$

先选择菜单 1，对 $M_1$ 查询 tag；再选择菜单 2，提交未查询过的 $M_2$ 与同一 tag：

```python
r.sendline(b'1')
r.sendline(m1.hex().encode())
tag = r.recvline_contains(b'Tag:').split()[-1]

r.sendline(b'2')
r.sendline(m2.hex().encode())
r.sendline(tag)
```

集合 `QUERIED` 按消息字节判重，故 $M_2$ 能通过“未查询”检查；MD5 状态碰撞又使 `nmac(M2) == tag`。服务端返回：

```text
tjctf{i_pr0baBly_sh0Uldnt_r0LL_my_0wn_M4CS}
```

## 方法总结

- `hash(message || key)` 不受普通长度扩展影响，不代表它是安全 MAC；底层哈希的碰撞仍可在共同后缀下延续。
- 攻击要求碰撞消息等长并在完整分组边界汇合内部状态，任意两段同摘要文本并不自动满足这一性质。
- 实际系统应使用 HMAC 或经过标准分析的 MAC，而不是调整密钥拼接位置。
