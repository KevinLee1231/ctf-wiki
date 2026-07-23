# Introduction to C

## 题目简述

题目给出一张看似随机噪点的 PNG 和一份颜色映射文本。图片中的每个像素实际编码一个 ASCII 字符，按行恢复后得到一份 C 源码；编译或继续阅读源码即可解出 flag。

## 解题过程

生成器对 ASCII 值 $i$ 使用以下颜色映射：

$$
(r,g,b)=
\bigl(722i\bmod256,\ 126i\bmod256,\ 15i\bmod256\bigr).
$$

先为 $0$ 到 $127$ 建立反向字典，再按从上到下、从左到右的顺序遍历图片：

```python
from PIL import Image

color_to_ascii = {}
for i in range(128):
    color = ((i * 722) % 256, (i * 126) % 256, (i * 15) % 256)
    color_to_ascii[color] = chr(i)

image = Image.open("intro_to_c.png").convert("RGB")
source = "".join(
    color_to_ascii[image.getpixel((x, y))]
    for y in range(image.height)
    for x in range(image.width)
)
```

输出是一段完整 C 程序。它遍历常量数组并计算：

```c
undecyphered_char = (char)((numbers[i] - 4563246763) ^ 97);
```

可以直接用 `gcc` 编译恢复出的源码，也可以在 Python 中逐项执行同一算式。最终输出：

```text
UMDCTF{pu61ic_st@t1c_v0ID_m81n_s7r1ng_@rgs[]!!!}
```

原始 PNG 只是逐像素数据容器，保留后视觉上仍是无意义噪点，因此归档 WP 用公式和解码代码完整替代了图片截图。

## 方法总结

- 核心技巧：把 RGB 三元组视为字符码表，而不是尝试常规图像隐写工具。
- 图片遍历顺序必须与生成器一致；交换 $x/y$ 或改变行序会打乱源码。
- 颜色值由模运算精确生成，应使用 RGB 原值，避免缩放、压缩或颜色管理改变像素。
- 恢复出源码后仍需阅读最终的减法与异或逻辑，不能只停在“得到 C 文件”。
