---
type: technique
tags: [reverse, font, glyph, text-rendering, texture]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
  - ../raw/reverse/D3CTF2019-c-c-wp.md
  - ../raw/reverse/UMDCTF2023-machokes-flex-wp.md
updated: 2026-07-28
---

# Font, Glyph and Text-Rendering Reconstruction

## 适用场景

秘密由自定义字体、私用区字符、glyph atlas、纹理 ID 或离屏文字渲染承载；关键是恢复字符到字形/纹理的映射以及实际绘制坐标，而不是只搜索普通字符串。

## 识别信号

- 文本包含 `U+E000` 等私用区字符，必须依赖题目字体才能显示。
- 程序调用 FreeType、`AddFontMemResourceEx` 或构建 glyph texture atlas。
- 硬编码纹理 ID、字符索引或离屏顶点按程序内创建顺序映射为文字。
- 静态字体包是占位/加密数据，运行时 hook 会替换真实字体内存。

## 最小证据

- 找到实际加载的字体/glyph 数据和字符映射来源。
- 用一个已知字符或 flag 前缀验证 glyph、texture ID 与字符的对应关系。
- 说明坐标系、atlas 顺序、fallback 和离屏/裁剪条件。

## 解法骨架

1. 跟踪字体加载、glyph rasterization 和纹理创建顺序。
2. 若字体运行时解密或替换，在真实 API 参数处 dump 最终样本。
3. 建立 codepoint/glyph index/texture ID 到字符或位图的映射。
4. 还原顶点位置、相机/投影和离屏文字，或直接导出 glyph 序列。
5. 用已知前缀和独立渲染验证映射，避免凭视觉猜测。

## 关键变体

- 私用区 codepoint 依赖自定义字体。
- FreeType glyph atlas 与 OpenGL texture ID。
- 加密字体包和 inline-hook 参数替换。
- 文本位于视锥外或 alpha/裁剪状态下。

## 常见陷阱

- 把纹理 ID 当通用字符编码；它通常只在当前创建顺序中有效。
- 只 dump 静态资源，忽略运行时替换后的字体。
- 忽略 fallback、kerning、Y 轴方向或相机裁剪。
- 把 shader 计算逻辑也塞进本页；自定义 shader VM 应独立模拟。

## 关联技巧

- [shader-vm-and-graphics-pipeline-emulation.md](shader-vm-and-graphics-pipeline-emulation.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)
- [game-asset-and-scene-state-extraction.md](game-asset-and-scene-state-extraction.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)

## 原始资料

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
- [D3CTF2019-c-c-wp](../raw/reverse/D3CTF2019-c-c-wp.md)
- [UMDCTF2023-machokes-flex-wp](../raw/reverse/UMDCTF2023-machokes-flex-wp.md)
