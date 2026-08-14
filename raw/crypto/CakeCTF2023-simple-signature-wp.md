# simple signature

## 题目简述

`crypto/simple_signature/challenge/server.py` 与 `distfiles/server.py` 组成签名题：

- 公钥参数：素数 $p$、基元 $g=2$，公开 `(w, v)`；
- 验签目标：
$$g^m \equiv s^w \cdot t^{-v}\pmod p$$
- 签名阶段禁止查询 `magic_word = "cake_does_not_eat_cat"` 的签名；
- `verify` 分支对目标消息若验证成功会直接输出 flag。

私钥生成保证 $v\equiv wy\pmod{p-1}$ 且 $u\equiv(wx-1)v^{-1}\pmod{p-1}$，于是 $wx-vu\equiv1$、$wy-v\equiv0$。诚实签名 $s=g^{xm+ry}$、$t=g^{um+r}$ 代入后正好得到 $g^m$。

问题在于验签方程只约束两个公开可控群元素，却没有要求它们来自签名算法。知道 $(w,v,p)$ 后，可以直接解指数线性同余，为任意目标 $m$ 构造 `(s,t)`；决定性障碍是代数伪造，归入 `crypto`。

## 解题过程

`solution/solve.py` 的核心步骤是把验证方程变成一次线性同余：

1. 连接服务并读取 `p`、`vkey=(w, v)`；
2. 计算目标消息哈希：
$$m = \mathrm{SHA512}(\text{magic\_word})$$
3. 取任意 $a=10$，求
$$a\cdot w - b\cdot v \equiv m \pmod{p-1}$$
即
$$b=(a w - m)\cdot v^{-1}\bmod(p-1)$$
4. 令
$$s=g^a\bmod p,\quad t=g^b\bmod p$$
并提交 `V` 流程。代入上式可得：
$$s^w\cdot t^{-v}\equiv g^{aw-bv}\equiv g^m\pmod p$$
因此验证成立。
5. 因提交消息为 `magic_word`，服务端进入 flag 分支。

```python
"""来自 solution/solve.py 的关键片段"""
magic_word = "cake_does_not_eat_cat"
m = h(magic_word)

a = 10
b = (a*w - m) * pow(v, -1, p-1) % (p-1)

s = pow(g, a, p)
t = pow(g, b, p)

sock.sendlineafter("erify: ", "V")
sock.sendlineafter("message: ", magic_word)
sock.sendlineafter("s: ", str(s))
sock.sendlineafter("t: ", str(t))
```

这里 $v$ 在生成时被强制满足 $\gcd(v,p-1)=1$，所以逆元一定存在。服务还要求 $2\le s,t<p$；官方选择的 $a=10$ 在实际实例中满足该范围，构造后仍应在提交前检查边界。

验证成功后得到：

```text
CakeCTF{does_yoshiking_eat_cake_or_cat?}
```

## 方法总结

- 核心技巧：将验证方程转成一元线性同余，直接构造 `(s,t)`，不需要调用签名服务。
- 识别信号：出现 $g^m=s^w t^{-v}\bmod p$ 这类公开指数关系，而签名由两个自由群元素组成时，应先在指数空间检查自由度是否足以直接解同余。
- 复用要点：指数等式在 $\mathbb{F}_p^*$ 中按模 $p-1$ 处理；先确认系数可逆，再验证构造元素满足服务端的数值范围。签名接口拒绝目标消息并不能弥补验证方程本身缺少不可伪造约束的问题。
