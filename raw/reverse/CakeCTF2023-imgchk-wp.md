# imgchk

## 题目简述

题目是“图像列 hash 校验”题：程序读取一张 `480 x 20` 的 PNG（`PNG_COLOR_TYPE_GRAY` 且 `bit_depth=1`）后，把每一列视为一段 20-bit 数据，取 MD5 与预置答案数组逐列比对。

`challenge/main.c` 的关键流程如下：

- 读取图片并要求尺寸与格式匹配；
- 对每列 `x`，按像素列读取 20 行的 bit，按 `(y % 8)` 组装成 3 字节缓冲；
- `MD5(buf, 3) == answer[x]` 全部为真才返回通过；
- 不足一列或任一列不匹配即 `Wrong...`。

官方 `gen.py` 给出另一条可行构造路径：直接把 `flag` 绘制成同尺寸、同字体、同通道的位图，再保存为 `flag.png`。

## 解题过程

### 关键观察

这个题的“可验证对象”是图片像素列本身，不是文本输入，也不是 hash 直接碰撞。

源码给出的两条证据链很关键：

1. `solution/solve.py` 在 `distfiles/imgchk` 的 `0x6020` 处读取一个由 480 个 8 字节值组成的指针表；每个表项指向对应列的 16 字节 MD5 摘要。
2. `challenge/gen.py` 说明这些列来自一张把 flag 文本居中绘制到 $480\times20$ 单色画布上的图片。

### 求解步骤

官方解法从发布给选手的二进制中恢复目标图，而不依赖仓库私有的原始 `flag.png`：

1. 对第 $x$ 列，从文件偏移 `0x6020 + 8x` 读取一个 8 字节小端地址，再到该地址读取 16 字节目标 MD5。
2. 一列只有 20 个像素，完整状态空间为 $2^{20}$。枚举 $v\in[0,2^{20})$，计算 `MD5(v.to_bytes(3, "little"))`，命中目标摘要后即可得到该列。
3. 将 $v$ 的第 $i$ 位写回坐标 $(x,i)$。相同摘要直接复用缓存，避免对空白列等重复图案反复枚举。
4. 处理完 480 列后保存 `output.png`，肉眼读取图中的 flag。

官方脚本的核心逻辑如下：

```python
known = {}
for x in range(480):
    target = md5s[x]
    if target not in known:
        for v in range((1 << 20) - 1, -1, -1):
            if hashlib.md5(v.to_bytes(3, "little")).digest() == target:
                known[target] = v
                break

    v = known[target]
    for y in range(20):
        img.putpixel((x, y), (v >> y) & 1)
```

`challenge/gen.py` 则可用于交叉验证位序、字体和画布参数：它确实以同样尺寸和 1-bit 模式绘制文本，因此逆出的列能重新通过校验程序。

### 验证

`task.yml` 明确给出的 flag 为 `CakeCTF{fd408e00d5824d7220c4d624f894144e}`，`gen.py` 产物与验证逻辑一致，不需要额外猜测。

## 方法总结

- 核心技巧：把二进制判定降为“列级位序 + MD5 匹配”。
- 识别信号：固定分辨率/位深、逐列 hash、可得预置答案表；优先找可逆生成链（`gen.py`）再判断是否需二次破解。
- 复用要点：不要把“摘要不可逆”误解成必须碰撞 MD5。单列输入只有 20 bit，可直接穷举原像；同时要核对三字节小端组包和纵向 bit 顺序。
