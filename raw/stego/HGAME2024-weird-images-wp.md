# 奇怪的图片

## 题目简述

附件是一组尺寸相同、外观高度相似的 RGB 图片。每张图包含不同的像素扰动；将两张图按通道逐像素异或时，公共背景会抵消，差异区域会显出一个 flag 字符。多次更换配对即可恢复完整 flag。

官方 PDF 把本题放在 Crypto，但真正决定解法的是从图像载体中定位隐藏视觉差异，因此归入 Stego。

## 解题过程

先选择一张中间位置的图片作为参考图，不优先使用第一张和最后一张；再将它依次与其余图片执行 RGB 通道 XOR。只要某一对图片的背景构造相匹配，输出中就只剩一个清晰字符。

```python
from pathlib import Path

from PIL import Image

SOURCE = Path("png_out")
OUTPUT = Path("xor_results")
OUTPUT.mkdir(exist_ok=True)

images = sorted(SOURCE.glob("*.png"))
if len(images) < 2:
    raise ValueError("至少需要两张图片")

# 选中间图片，避免题解提示中不适合作参考的首尾样本。
reference_index = len(images) // 2
reference_path = images[reference_index]
reference = Image.open(reference_path).convert("RGB")

for candidate_path in images:
    if candidate_path == reference_path:
        continue

    candidate = Image.open(candidate_path).convert("RGB")
    if candidate.size != reference.size:
        raise ValueError(f"尺寸不一致：{candidate_path.name}")

    result = Image.new("RGB", reference.size)
    ref_pixels = reference.load()
    candidate_pixels = candidate.load()
    result_pixels = result.load()

    for x in range(reference.width):
        for y in range(reference.height):
            r1, g1, b1 = ref_pixels[x, y]
            r2, g2, b2 = candidate_pixels[x, y]
            result_pixels[x, y] = (r1 ^ r2, g1 ^ g2, b1 ^ b2)

    result.save(OUTPUT / f"xor-{reference_path.stem}-{candidate_path.stem}.png")
```

如果当前参考图不能覆盖全部字符，就换另一张非首尾图片重复处理。检查输出图时记录每个清晰字符及其顺序，最终拼接为完整 flag。

PDF 只展示了处理思路与代码，没有嵌入原始题目图片或最终 flag；因此这里不凭空补造结果，也没有创建无来源的图片资源目录。

## 方法总结

- 核心技巧：对同尺寸图像执行逐像素 XOR，让相同背景抵消并突出隐藏差异。
- 识别信号：大量几乎相同的图片、单图观察没有信息、题目暗示“差异”或需要交叉比较。
- 复用要点：先统一颜色模式和尺寸；不同参考图可能只揭示部分字符，应保存“参考图—候选图—字符”的对应关系，避免恢复后无法确定顺序。
