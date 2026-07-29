# EZMaze

## 题目简述

服务随机生成一条长度为 64 的迷宫路径，把 `L、R、U、D` 分别编码为二进制 `00、01、10、11`，再用 RSA 加密对应的 128 位整数。客户端不能直接解密，但可以提交任意密文；服务会把解密结果重新解释成路径，只返回终点坐标。

终点坐标看似只泄露两个计数差，实际能给出路径长度的奇偶性。结合 RSA 的乘法同态，可以把它改造成区间判定预言机，并二分恢复原始路径整数。

## 解题过程

路径编码等价于把每一步当作一个四进制数字：

```python
directions = "LRUD"

def toInt(path):
    value = 0
    for direction in path:
        value = value * 4 + directions.index(direction)
    return value
```

服务返回：

```python
(count("R") - count("L"),
 count("U") - count("D"))
```

每一步对两个坐标之和的贡献都是奇数，因此：

$$
(x+y)\bmod 2=\lvert\operatorname{toPath}(m)\rvert\bmod 2.
$$

设：

$$
B=2^{1022}=4^{511}.
$$

若能保证 RSA 解密结果 $r$ 落在 $[B/4,2B)$，则：

- $B/4\le r<B$ 时，$r$ 有 511 个四进制位，坐标和为奇数；
- $B\le r<2B$ 时，$r$ 有 512 个四进制位，坐标和为偶数。

于是坐标奇偶性就等价于判断 $r<B$。

RSA 满足乘法同态。对目标密文 $c=m^e\bmod n$，提交：

$$
c'=c\cdot f^e\bmod n,
$$

服务实际解密出：

$$
r=mf\bmod n.
$$

维护候选区间 $m\in[L,H]$，选择整数 $i$ 和乘数 $f$，让整个区间经乘法后落入：

$$
i n+B/4\le Lf\le Hf\le i n+2B.
$$

此时预言机返回的奇偶性可以判断 $mf-in$ 位于 $B$ 左侧还是右侧，从而把 $[L,H]$ 二分。官方脚本的核心为：

```python
def get_info(f):
    crafted = enc_sol * pow(f, e, n) % n
    x, y = query_decryption(crafted)
    return "<" if (x + y) % 2 else ">"

while high - low > 100:
    tmp = 3 * B // 2 // (high - low)
    i = (tmp * low - B // 4) // n
    mid = (low + high) // 2
    f = (i * n + B) // mid

    if get_info(f) == "<":
        high = mid
    else:
        low = mid + 1
```

长度为 64 的路径对应至多 128 位整数。脚本从 $[2^{127},2^{128}]$ 开始，因此只覆盖最高位为 1 的实例，概率约为 $1/2$；若断言失败或最终区间无解，重新连接获取一条新路径即可。

当区间足够小时，逐个检查：

```python
while low <= high and pow(low, e, n) != enc_sol:
    low += 1
```

找到唯一明文整数后按每 2 位映射回 `LRUD`，提交完整路径，得到：

```text
SEKAI{parity_reveals_everything_:<_8f1261a517796b4d}
```

## 方法总结

预言机泄露的信息不必直接是明文位。只要输出与明文所在区间存在稳定关系，就能借助 RSA 乘法同态主动移动明文，再反复缩小范围。本题的关键是先证明坐标和奇偶性等于四进制位数奇偶性，再把查询结果限制在 $[B/4,2B)$，从而得到可靠的阈值判断。
