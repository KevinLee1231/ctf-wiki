# GreyCTF2023 GreyCat Trial

## 题目简述

服务要求提交整数 $a,b$，随后随机取 $n\in[0,26]$ 并令 $c=a+bn$。程序只用

`pow(23456789, c - 1, c) == 1`

判断 $c$ 是否“像素数”，通过即给出 flag。这里使用的是单一底数的 Fermat 测试，并不能证明 $c$ 为素数；更重要的是，随机挑战只有 27 个候选值。

## 解题过程

目标是构造一段等差数列，使尽可能多的 $a+bn$ 都能通过检查。官方解法采用已知的长素数等差数列：

```python
a = 3486107472997423
b = 371891575525470
```

其中前 26 项均为素数，因此对底数 `23456789` 必然满足 Fermat 同余。服务抽到这些位置时直接成功；若抽到未覆盖的一项，重新连接即可。交互只需提交上述两数并检查响应：

```python
while True:
    io = connect()
    io.sendlineafter(b"a = ", str(a).encode())
    io.sendlineafter(b"b = ", str(b).encode())
    out = io.recvall()
    if b"grey{" in out:
        print(out.decode())
        break
```

最终得到：

```text
grey{Gr33N-tA0_The0ReM_w1z4rd}
```

## 方法总结

本题有两层薄弱点：Fermat 单底测试只是必要条件，随机索引空间又很小。实际利用甚至不必构造复杂伪素数，只要让绝大多数候选项本身为素数，再以重连吸收剩余失败概率即可。面对概率型验证，应先枚举随机空间并估算重试成本，而不是默认必须一次通过。
