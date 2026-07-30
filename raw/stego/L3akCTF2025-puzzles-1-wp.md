# L3akCTF 2025 Puzzles 1 Writeup

## 题目简述

Puzzles 是连续五题，Puzzles 1 对应第一关。服务端从 10 张原图中依次选择图片，把每张图切成 $4\times4$ 共 16 块，随机打乱后通过 `/api/newpuzzle` 返回：

```json
{
  "title": "Classy Lady",
  "url": "https://原图来源",
  "pieces": ["base64 PNG", "..."],
  "rows": 4,
  "cols": 4,
  "puzzle_id": "..."
}
```

需要在每张图的时限内提交正确排列，并解完本关全部 10 张图。第一关规模很小，可以直接观察边缘连续性手工拼接；也可以利用响应中的标题和原图地址做像素匹配。决定性障碍是恢复图像碎片的二维顺序，因此归入 stego。

## 解题过程

### 理解答案数组

服务端切图时给每个原始位置分配行优先索引：

$$
i=y\cdot \text{cols}+x
$$

随后对切片执行 Fisher–Yates 洗牌。返回的 `pieces[k]` 表示打乱后位于索引 $k$ 的图片块；提交的 `answer[i]` 则应填写“原始位置 $i$ 对应的块目前位于返回数组的哪个索引”。

因此，若观察得到正确顺序是返回块 `[5, 2, 9, ...]`，提交体应为：

```json
{
  "puzzle_id": "当前 UUID",
  "answer": [5, 2, 9]
}
```

数组必须覆盖全部 16 个位置。

### 手工拼接

$4\times4$ 的边界很容易识别：

1. 服务端给原图外侧增加一圈纯色边框，先找出具有两条纯色边的四个角；
2. 具有一条纯色边的是其余外框块；
3. 比较相邻块的右边缘与左边缘、下边缘与上边缘；
4. 先完成外框，再填内部 4 块；
5. 在网页中按该顺序拖动并提交。

依次完成 10 张图后即可达到本关 `solves_needed = 10`。

### 使用原图自动匹配

API 同时返回图片标题和来源地址，因此可以下载未切分原图。仓库公开后，也可直接使用 `build/images/level1` 中的同一批图片。

服务端预处理逻辑是：

- 边框宽度为 5 像素；
- 裁剪后的宽高与 $4\times4$ 网格整除；
- 外侧填充随机浅色；
- 再按行优先顺序切片。

本地复现切图时可使用透明边框。评分函数遇到“本地透明、远端浅色”的外框像素时忽略颜色差异，对图像主体则计算抽样像素的 RGB 曼哈顿距离：

$$
d(A,B)=\sum_{p}\left(
|A_p^R-B_p^R|+
|A_p^G-B_p^G|+
|A_p^B-B_p^B|
\right)
$$

核心的一对一匹配逻辑如下：

```python
def distance(left, right):
    total = 0
    for p1, p2 in zip(left, right):
        if p1[3] < 10:
            if min(p2[:3]) < 127:
                total += 1000
        elif p2[3] < 10:
            if min(p1[:3]) < 127:
                total += 1000
        else:
            total += sum(abs(a - b) for a, b in zip(p1[:3], p2[:3]))
    return total


remaining = set(range(len(received_fingerprints)))
answer = []

for reference in reference_fingerprints:
    found = min(
        remaining,
        key=lambda index: distance(reference, received_fingerprints[index]),
    )
    answer.append(found)
    remaining.remove(found)
```

每匹配一个原始块，就从候选集合删除对应远端块，避免多个位置错误地复用同一切片。将得到的 16 项数组提交到 `/api/checkanswer`，成功响应中的 `correct` 为 `true`。

完成 10 张图后，通过 `/api/getflag` 请求零基关卡编号 `0`：

```json
{"level": 0}
```

得到：

```text
L3AK{1_th4t_w45_pr3tty_34sy}
```

## 方法总结

第一关既可视为普通小型拼图，也展示了后续关卡的通用数据模型：答案不是原始块编号列表，而是从原始位置到返回数组索引的逆置换。先确认这一点，可以避免图像已经拼对却因数组方向相反而提交失败。

自动化时不必逐像素比较全部图片。对每块稀疏抽样并做一对一最小距离匹配，在 $4\times4$ 规模下已经足够稳定；随机边框则应按“外框掩码”处理，而不能要求本地颜色与服务端完全相同。
