# pseudo-brainrot

## 题目简述

题目把 36 字节 flag 展开为 288 bit，先用一个置换打乱位序，再把每一位写入伪随机像素的伪随机 RGB 通道最低位。附件同时给出完整 `encode.py`，所有随机过程最终由固定种子 42 决定；大量无意义的 `troll = random.randint(...)` 只用于扰乱对 PRNG 状态的跟踪。

## 解题过程

Python `random` 是确定性状态机。先按生成器原顺序执行：

1. `seed(42)` 后生成 `lmao`，再以 `lmao` 重新播种；
2. 保留每一次看似无用的 `randint`，因为它们都会推进状态；
3. 重建长度 288 的 `inds` 洗牌结果；
4. 重建 288 个 `pic_inds` 和每位对应的 RGB 通道 `changes`；
5. 在生成器每逢 `i % randnum == 0` 重新播种的位置做同样操作。

官方 `solution.py` 完整复刻了这些调用。得到三个数组后，实际提取逻辑很短：

```python
from PIL import Image

image = Image.open("skibidi_encoded.png")
width, height = image.size

permuted = []
for pixel_index, channel in zip(pic_inds, changes):
    row = pixel_index // height
    col = pixel_index % width
    permuted.append(image.getpixel((row, col))[channel] & 1)

original = [0] * 288
for output_position, original_position in enumerate(inds):
    original[original_position] = permuted[output_position]

flag = bytes(
    int("".join(map(str, original[i:i + 8])), 2)
    for i in range(0, 288, 8)
)
print(flag.decode())
```

恢复结果为：

```text
tjctf{th@t_m@d3_m3_g0_1n5an3333!!!!}
```

编码前后图片肉眼几乎相同，载体画面本身不提供额外线索，因此 WP 不重复保存两张视觉等价的梗图。

## 方法总结

- 已知固定种子的伪随机位置不是秘密；只要精确重放调用序列，就能恢复所有像素和通道位置。
- 无用随机调用仍会改变后续输出，删除任何一个都会让整条序列错位；应把它们当状态转换，而不是当可忽略的死代码。
- 提取后还需应用逆置换：编码是 `new[i]=old[inds[i]]`，解码应写回 `old[inds[i]]=new[i]`。
