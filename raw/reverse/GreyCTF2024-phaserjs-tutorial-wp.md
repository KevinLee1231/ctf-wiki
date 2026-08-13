# PhaserJS Tutorial

## 题目简述

网页游戏把 64 字节状态连续变换一千万轮。每轮依次执行带反馈的异或、位置置换和 RSA 私钥指数运算；官方附件给出质因数后，可以按相反顺序逐轮求逆，最终从 64 字节状态的后 48 字节恢复 flag。游戏素材只是装饰，不参与解题，因此无需保留截图或精灵图。

## 解题过程

单轮正向过程可概括为：

1. 从索引 24 开始，以 `wave & 0xff` 为初值做链式异或；
2. 对每个索引 $i$，与 $((i\times wave)\bmod40)+24$ 交换；
3. 把 64 字节视为大端整数，计算 $n^d\bmod m$。

模数为 $m=pq$，公开指数 $e=65537$。逆变换必须倒序处理三个步骤：先计算 $n^e\bmod m$，再从索引 63 反向撤销交换，最后恢复异或反馈。核心实现为：

```python
def untransform(bs, wave, m, e=65537):
    n = int.from_bytes(bytes(bs), "big")
    bs = list(pow(n, e, m).to_bytes(64, "big"))

    for i in range(63, 23, -1):
        j = ((i * wave) % 40) + 24
        bs[i], bs[j] = bs[j], bs[i]

    k = wave & 0xff
    for i in range(24, 64):
        bs[i] ^= k
        k = bs[i]
    return bs
```

附件给出的两个质因数为：

```python
p = 2933342412243178360246913963653176924656287769470170577218737
q = 2663862733012296707089609302317500558193537358171126836499053
m = p * q
```

从脚本中的最终状态 `t` 开始，按波次 1 到 10000000 调用 `untransform`：

```python
for wave in range(1, 10_000_001):
    t = untransform(t, wave, m)
print(bytes(t)[16:].decode())
```

输出为：

```text
grey{ea_sports_its_in_the_game_586256cbd58140ec}
```

## 方法总结

复合变换求逆时，运算顺序与每个运算自身的逆都要同时反转。RSA 部分无需重新求私钥，因为正向使用私钥指数，逆向直接用公开指数；交换必须倒序回放，反馈异或则要在覆盖当前字节后把恢复值作为下一轮密钥。
