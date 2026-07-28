---
type: technique
tags: [reverse, shader, glsl, spir-v, gpu, emulation]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
updated: 2026-07-28
---

# Shader VM and Graphics-Pipeline Emulation

## 适用场景

校验或解码逻辑位于 GLSL/SPIR-V、fragment/vertex/compute shader 或自定义 graphics VM 中；需要恢复 uniform、buffer、纹理采样和并行 invocation 语义，再用真实 GPU 或 CPU harness 重放。

## 识别信号

- 程序加载 GLSL/SPIR-V、创建 SSBO/UBO/FBO 或把输入上传为纹理。
- 关键状态只在 draw/dispatch 后出现在 framebuffer、buffer 或像素中。
- shader 含自定义 opcode dispatch、轮函数、共享内存或 invocation 间索引关系。
- CPU 侧只负责资源绑定，真正比较/变换在 GPU 管线中。

## 最小证据

- 固定 shader stage、entry point、workgroup/draw 参数和资源 binding。
- 记录 uniform、buffer、texture format、坐标系与整数/浮点精度。
- 单个 invocation 或小输入的输出可与原程序/GPU capture 对齐。

## 解法骨架

1. 从 API trace 或反编译结果恢复 pipeline state 和所有资源 binding。
2. 将 shader 反编译为可读逻辑，标注 invocation ID、共享状态与位宽。
3. 优先写最小无窗口 harness 重放；GPU 环境不稳定时再移植为 CPU 模型。
4. 导出 framebuffer/SSBO 中间状态，定位真正校验或解码输出。
5. 用边界输入比较 GPU 与模型结果，处理浮点、wrap、采样和未定义行为差异。

## 关键变体

- Fragment shader 通过像素/纹理输出承载数据。
- Compute shader/SSBO 执行并行变换。
- SPIR-V 或自定义 shader bytecode 需要 lifting。
- Shader 内部再次实现 VM/SMC 状态机。

## 常见陷阱

- 忽略资源 binding 或 texture format，导致“逻辑正确、输出全错”。
- 把并行 invocation 当顺序循环，漏掉 barrier/shared memory。
- CPU 浮点与 GPU 精度/舍入不一致。
- 仅凭“程序用了文字 shader”就归入本页；普通 glyph 映射应转字体技巧。

## 关联技巧

- [font-glyph-and-text-rendering-reconstruction.md](font-glyph-and-text-rendering-reconstruction.md)
- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)
- [game-asset-and-scene-state-extraction.md](game-asset-and-scene-state-extraction.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)

## 原始资料

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
