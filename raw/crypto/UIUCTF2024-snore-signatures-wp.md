# Snore Signatures

## 题目简述

服务实现了一个改造过的 Schnorr 签名。每轮给出公钥 $y=g^{-x}$，允许对一个消息 $m$ 请求签名 $(s,e)$，随后必须在不能修改 $e$、也不能复用任何历史消息的条件下伪造签名。连续完成 10 轮才能取得 Flag。

## 解题过程

签名端计算

$$
R=g^k\bmod p,\qquad e=H((R+m)\bmod p)\bmod q,
$$

以及 $s=k+xe\pmod q$。验证端恢复的群元素为

$$
R_v=g^s y^e=g^{k+xe}g^{-xe}=g^k=R\pmod p.
$$

漏洞在于挑战值哈希的是整数加法 `R + m`，而不是无歧义拼接。令 $s'=s+1\pmod q$，验证端将恢复

$$
R'=g^{s'}y^e=Rg\pmod p.
$$

再选择 $m'=m+R-R'\pmod p$，便有 $R'+m'\equiv R+m\pmod p$，所以哈希结果仍是原来的 $e$。无需知道私钥或随机数即可完成伪造。

每轮交互的关键代码如下；`R_prime` 必须先按模 $p$ 计算，且极低概率出现 $s'=0$ 时应重新连接，因为验证器明确拒绝零：

```python
# 已读取本轮的 p, q, g, y，以及查询得到的 s, e。
R = (pow(g, s, p) * pow(y, e, p)) % p
s_prime = (s + 1) % q
if s_prime == 0:
    raise RuntimeError("reconnect and request another signature")

R_prime = (R * g) % p
m_prime = (m + R - R_prime) % p

assert m_prime != m
# 向服务依次提交 m_prime 与 s_prime；e 沿用服务锁定的原值。
```

为避免触发全局消息去重，查询消息可按轮次选择不同的小整数，并记录所有查询值与伪造值；若新值碰撞就重新选消息。连续通过 10 轮后得到：

```text
uiuctf{add1ti0n_i5_n0t_c0nc4t3n4ti0n}
```

## 方法总结

- Schnorr 的挑战应绑定承诺与消息的无歧义编码；把二者做可补偿的模加法会产生代数可塑性。
- 攻击链是先令响应 $s$ 增加，从而准确预测验证端承诺如何乘上 $g$，再反向调整消息抵消变化。
- 实现时还要满足协议边界：$0<s'<q$、新消息不能与任一历史消息重复，并应先验证 $(R'+m')\bmod p=(R+m)\bmod p$ 再提交。
