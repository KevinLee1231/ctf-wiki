# UIUCTF 2023 futuredisk2 Writeup

## 题目简述

第二题仍提供一个接近 8 EiB、支持 HTTP Range 的虚拟 gzip 文件，但背景压缩流不再由等长块简单重复。每个 DEFLATE match 块的 match 数依次变化，必须先建立“任意位偏移处应该出现什么字节”的数学模型，才能把远程样本与预测值比较并二分 flag。

## 解题过程

### 还原嵌套块序列

用 Range 下载文件开头并按 DEFLATE 位流分析，可见一个含 $n$ 个 match 的块长度为：

$$
L(n)=2n+100\text{ bits}.
$$

块数按照三角形序列排列：

```text
3, 4, 5, ..., 32767
   4, 5, ..., 32767
      5, ..., 32767
...
32767
```

到末行后又从 3 开始。整个大周期的位长为：

$$
M=\sum_{r=3}^{32767}\sum_{n=r}^{32767}(2n+100)
 =23506705809710.
$$

在偏移 $M$ 的整数倍附近做 Range 请求，可以观察到 `32766, 32767, 3, 4, 5` 的重置，从而验证模型。

### 将位偏移映射到行、列和块内位置

给定去掉初始块后的位偏移 $B$，先计算：

$$
q=\left\lfloor\frac{B}{M}\right\rfloor,
\qquad b=B\bmod M.
$$

从第 3 行累计到第 $x$ 行的长度可写成三次多项式：

$$
F(x)=-\frac{x^3}{3}-50x^2+\frac{3230957419}{3}x-2153971410.
$$

求满足 $F(x)\le b<F(x+1)$ 的最大整数 $x$，即可确定行号；把该行之前的长度减掉后，行内从块 $r$ 累计到块 $y$ 的长度是二次式：

$$
G_r(y)=y^2+101y+100-r^2-99r.
$$

再求 $G_r(y)\le b'<G_r(y+1)$，即可得到当前块的 $n=y+1$ 和块内剩余位数。实现时可以用多项式求根得到近似整数，再在邻近 $\pm2$ 范围逐项校正，避免浮点取整落到相邻块。

有了 $(q,r,n,\text{remainder})$，本地生成同尺寸 match 块，再跳过 `remainder` 位，就能预测任意远程偏移的若干字节：

```python
def expected_at(bit_offset):
    (_, row, n), remainder = invert_sequence(bit_offset % M)
    return gen_match_chunk(n).skip_bits(remainder)
```

### 用预测值做二分

选择一个已知在 flag 前的字节偏移 `left` 和一个靠近文件尾、已知在 flag 后的 `right`。对中点 `mid`：

1. 由多项式模型生成 `mid` 处以及下一块的预测字节；
2. 用 `Range: bytes=mid-...` 获取同长度真实数据；
3. 两者相同则中点仍在 flag 前，令 `left = mid`；不同则已受插入块影响，令 `right = mid`。

仓库求解器从头计算时二分到的候选字节偏移为：

```text
6256375869413382493
```

在该位置下载约 8205 字节，异常 DEFLATE 数据位于窗口内约 7850 字节处。和第一题一样，块起点可能不按字节对齐，因此从该处逐位调用 `skipBits()`，以 `wbits=-zlib.MAX_WBITS` 尝试 raw DEFLATE 解压，并筛选包含 `uiuctf{` 的结果：

```text
uiuctf{i sincerely hope that was not too contrived, deflate streams are cool}
```

## 方法总结

第二题把“固定周期检测”升级为“可计算的非均匀周期检测”。关键不是存储或遍历 8 EiB 数据，而是建立偏移到生成状态的反函数：先对完整序列取模，再解三次式定位行、解二次式定位块，最后生成少量预测字节。HTTP Range 仍是决定性 Web 原语，DEFLATE 数学模型则负责构造二分所需的前后判定。
