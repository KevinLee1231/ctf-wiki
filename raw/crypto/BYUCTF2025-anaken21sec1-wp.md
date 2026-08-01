# Anaken21sec1

## 题目简述

题目给出一套自定义分组密码及已知密钥 `orygwktcjpb`，要求还原密文。算法先过滤非字母字符，并把明文用 `x` 补齐到 12 字符的倍数。每个 12 字符分组被编码为一个 $6\times6$ 的三进制矩阵；随后对密钥的 11 个字母依次执行位置置换和模 3 加法，最后再按密钥字母的字典序重排所有输出字符。

关键点是所有步骤都可逆：五张置换表都是 $1\ldots36$ 的排列，而加法只发生在模 3 环上。

## 解题过程

先逆转末尾的列式重排。加密器把中间字符串按位置模 11 分入 11 个盒子，再按密钥字母从小到大依次拼接。已知密钥后，可以计算每个盒子的长度，按相同字典序从密文中切片，最后轮询各盒子恢复原来的中间字符串。

对每个 12 字符分组，按加密器相反的方式重建矩阵。字符 `0` 表示三进制值 0，其余字符 `a` 到 `z` 对应 1 到 26。随后逆序遍历密钥：先撤销加法，再撤销置换。模 3 下 $-1\equiv2$，因此原来的 `dst += src` 可用 `dst += 2*src` 撤销。

核心逆变换如下：

```python
def unpermute(block, table):
    out = np.zeros((6, 6), dtype=int)
    for i in range(6):
        for j in range(6):
            pos = int(table[i, j] - 1)
            out[pos // 6, pos % 6] = block[i, j]
    return out

def unadd(block, mode):
    if mode == 0:
        for i in range(6):
            for j in range(6):
                if (i + j) % 2 == 0:
                    block[i, j] += 2
    elif mode == 1:
        block[3:, 3:] += 2 * block[:3, :3]
    elif mode == 2:
        block[:3, :3] += 2 * block[3:, 3:]
    elif mode == 3:
        block[3:, :3] += 2 * block[:3, 3:]
    else:
        block[:3, 3:] += 2 * block[3:, :3]
    return block % 3
```

将所有轮次反演并把三进制位重新组合成字母，去掉末尾填充后得到：

```text
byuctf{revisreallythestartingpointformostcategoriesiydk}
```

## 方法总结

- 核心技巧：逐层逆转“全局重排—轮函数—字符编码”，并在模 3 环中用加 2 撤销加 1。
- 识别信号：自定义密码若只由排列、模加法和固定进制编码组成，通常无需密码分析，直接构造严格逆函数即可。
- 复用要点：复合变换必须按相反顺序撤销；置换的逆映射应写回原索引，而不是再次套用原表。
