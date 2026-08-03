# the shortest crypto chal

## 题目简述

整个挑战逻辑压缩在一条断言中。需要寻找小于 $20000$ 的整数 $a,b,c,d$，满足：

$$
a^4+b^4=c^4+d^4+17.
$$

四个数的乘积转成十进制并左侧补零到 16 字节，作为 AES-128-ECB 密钥；flag 本身长度恰为两个分组，因此加密前没有 padding。真正的难点是从约 $20000^4$ 的朴素四元组空间中找到相差 17 的两个四次幂和。

## 解题过程

把问题拆成两个二元和：

$$
S(x,y)=x^4+y^4.
$$

若先保存全部 $S(x,y)$ 再排序，需要 $O(H^2)$ 内存。仓库 solver 使用最小堆，把每个固定 $y$ 对应的单调序列：

$$
2^4+y^4,3^4+y^4,\ldots,(H-1)^4+y^4
$$

做多路归并。堆里每个 $y$ 只保留当前最小项，因此内存为 $O(H)$；弹出一项后再压入同一列的下一项。只保留 $x<y$ 可以去掉交换对称。

```python
import heapq

H = 20000
values = list(range(2, H))

def sorted_sums():
    heap = [(2**4 + y**4, 2, y) for y in values]
    heapq.heapify(heap)

    while heap:
        total, x, y = heapq.heappop(heap)
        if x < y:
            yield total, x, y
        if x + 1 < H:
            nx = x + 1
            heapq.heappush(heap, (nx**4 + y**4, nx, y))

previous = None
for current in sorted_sums():
    if previous is not None and current[0] - previous[0] == 17:
        low, c, d = previous
        high, a, b = current
        print(a, b, c, d)
        break
    previous = current
```

官方搜索在有序流中找到一对相邻和：

```text
a = 264
b = 651
c = 530
d = 570
```

直接验证：

$$
264^4+651^4-530^4-570^4=17.
$$

这里的“检查相邻项”并不是对任意偏移都完备的通用证明；它依赖本题目标解恰好在排序后的和集合中相邻。找到候选后必须回到原断言和密文做最终验证。

密钥为：

```python
key = f"{264 * 651 * 530 * 570}".zfill(16).encode()
assert key == b"0000051920114400"
```

用该密钥解密仓库给出的两个 AES 分组，或直接重新加密候选 flag 对照断言中的十六进制密文，可确认：

```text
uiuctf{D1oPh4nTine__Destr0yer__}
```

## 方法总结

- 核心技巧：把四变量丢番图方程改写成两个二元和的碰撞/固定差问题，再用堆归并按序枚举二元和。
- 识别信号：形如 $f(a)+f(b)=f(c)+f(d)+\Delta$、变量上界中等、无法承受完整二元表内存，但每一列按变量单调。
- 复用要点：多路归并把内存从 $O(H^2)$ 降为 $O(H)$，时间上界仍接近 $O(H^2\log H)$；任何提前终止条件都应说明完备性边界，并用原方程和密码学输出验证候选。
