---
type: technique
tags: [reverse, font, shader, renderer, legacy-format]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
updated: 2026-07-27
---

# Renderer, Glyph, Shader and Legacy-Format Reconstruction

## 适用场景

秘密由字体 glyph、shader、bitmap/legacy 文件格式或渲染管线生成；关键不是普通字符串搜索，而是恢复坐标、变换、字形映射和输出像素。

## 识别信号

- 字体表、自定义 glyph、GLSL/SPIR-V、tile/bitmap/MBR 等结构承载逻辑。
- 程序输出依赖顶点/fragment 变换、字符到轮廓映射或古老图像格式。
- 静态数据不可读，但经正确 renderer 后形成文字/图形。

## 最小证据

- 明确格式版本、坐标系、色彩/bit order 和渲染入口。
- 提取单个 glyph/像素/着色器输出验证解析。
- 最终图像可由独立 renderer 或参考实现复现。

## 解法骨架

1. 解析容器表和索引，定位 glyph/program/bitmap 数据。
2. 还原坐标、矩阵、采样和像素打包规则。
3. 写最小离屏 renderer 或转换器生成标准 PNG/文本。
4. 对照原程序输出和边界样本验证。

## 关键变体

- Font glyph/cmap/outline。
- Shader bytecode 与 GPU 变换。
- MBR/legacy bitmap/tile renderer。

## 常见陷阱

- 坐标原点、Y 轴方向或 alpha 混合错误。
- 用截图猜测而未解析原始结构。
- 忽略字体 fallback/kerning 或 shader uniform。

## 关联技巧

- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)
- [game-asset-and-scene-state-extraction.md](game-asset-and-scene-state-extraction.md)
- [unknown-isa-bootloader-and-firmware-emulation.md](unknown-isa-bootloader-and-firmware-emulation.md)

## 原始资料

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
