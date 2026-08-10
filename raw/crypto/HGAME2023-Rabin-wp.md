# Rabin

## 题目简述

题目给出 Rabin 密码的两个素因子 $p,q$ 和密文 $c$。加密关系为

$$
c\equiv m^2\pmod n,\qquad n=pq.
$$

需要分别求出模 $p$、模 $q$ 的平方根，再用中国剩余定理组合出四个候选明文，从中识别 flag。

```python
p = 65428327184555679690730137432886407240184329534772421373193521144693375074983
q = 98570810268705084987524975482323456006480531917292601799256241458681800554123
c = int(
    "4e072f435cbffbd3520a283b3944ac988b98fb19e723d1bd02ad7e58d9f01b26"
    "d622edea5ee538b2f603d5bf785b0427de27ad5c76c656dbd9435d3a4a7cf556",
    16,
)
```

## 解题过程

这里 $p\equiv q\equiv3\pmod4$。对于这种素数，$c$ 的一个平方根可直接写为：

$$
r_p\equiv c^{(p+1)/4}\pmod p,
$$

另一个根是 $-r_p\bmod p$；模 $q$ 同理。两边各有两个根，笛卡尔积一共形成四组同余方程：

$$
m\equiv x_p\pmod p,\qquad m\equiv x_q\pmod q.
$$

逐组使用中国剩余定理合并并转成字节：

```python
n = p * q
rp = pow(c, (p + 1) // 4, p)
rq = pow(c, (q + 1) // 4, q)
p_inv_mod_q = pow(p, -1, q)

for xp in (rp, -rp % p):
    for xq in (rq, -rq % q):
        m = (xp + p * ((xq - xp) * p_inv_mod_q % q)) % n
        raw = m.to_bytes(max(1, (m.bit_length() + 7) // 8), "big")
        if b"hgame{" in raw:
            print(raw.decode())
```

四个平方根中只有一个具有正确的 flag 格式：

```text
hgame{That'5_s0_3asy_to_s@lve_r@bin}
```

## 方法总结

Rabin 解密天然存在四个候选根。已知分解 $n=pq$ 后，先分别在两个素数域中开平方，再用 CRT 组合；最后依靠编码、填充或本题的 flag 格式判断正确明文。只计算一组正根会漏掉其余候选，是这类题最常见的错误。
