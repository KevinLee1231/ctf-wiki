---
type: tooling
tags: [ai-ml, tooling, tools, environment]
skills: [ctf-ai-ml]
updated: 2026-07-06
---

# AI/ML Tooling

本页记录 `ctf-ai-ml` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 工具选择边界

### 入口选择

- 首轮只依赖 `ctf-tools` 中已经可用的 `numpy`、`scipy`、`Pillow`、`scikit-learn`，避免为不确定题型先安装重量级框架。
- 需要深度学习框架时，先确认题目确实给出模型权重、推理接口、adapter、embedding 或训练数据，再按需补装。
- 如果只是 Web 上的模型服务、prompt/API 权限问题，先回到 `ctf-web` 或对应服务链路，不要直接装 ML 框架。

### 不应进入 AI/ML 工具链的情况

- 证据只有 HTTP 接口、登录态、API key、前端权限或模型服务路由时，先按 Web 处理。
- 证据只有自然语言 prompt、拒答、工具调用或上下文注入时，先看 [llm-attacks.md](llm-attacks.md)，不先安装 `torch`。
- 模型文件像压缩包、打包器、恶意载荷或混淆二进制时，先进入 Reverse/Malware 路线确认载体。

### 补工具经验的触发条件

- raw 给出 `.pt/.pth/.onnx/.safetensors/.tflite`，并且加载错误会影响解题。
- 需要比较权重、adapter、embedding 或 logits，且现有 `numpy/scipy/sklearn` 不足以复现。
- 远程 oracle 暴露概率、置信度、embedding 或 query budget，工具选择会影响可重复验证。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| `numpy` | 权重、向量、矩阵与 logits 的基础检查 |
| `scipy` | 统计分析与信号型特征 |
| `Pillow` | 图像输入和对抗样本的低成本检查 |
| `scikit-learn` | 阈值攻击、简单基线和评估指标 |

### 专项按需

- 当前已装的工具已经足够支撑“先判断问题类型”的首轮工作；只有确认进入深度模型分析后，再进入重量级依赖。

### 当前未装 / 建议按需补装

- `torch`
- `transformers`
- `safetensors`

这些都属于题型确认后的重量级依赖，适合继续保持“按需安装”，不要放进默认首轮环境。

## 失败信号与转向

- 只有 HTTP 接口、登录态、API key 或前端权限问题，没有模型证据：先转 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)，不要为了“看起来像 AI”安装深度学习框架。
- 查询接口不给概率、logits 或稳定标签：先构造少量代表性查询输入，确认反馈是否稳定可区分；如果只能得到自然语言拒答或工具调用，转 [llm-attacks.md](llm-attacks.md)。
- checkpoint、adapter 或 embedding 文件无法加载：先记录文件格式、magic、config 和版本错误；若像打包器/混淆载荷，转 [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md) 或 Reverse/Malware 页面。
- 需要 `torch`、`transformers` 或 `safetensors`：只有确认模型文件或推理代码确实依赖它们时再安装，并记录安装原因和复现实验。

## 详细清单

### ctf-tools conda 环境

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **numpy** | 2.4.4 | 数值/矩阵运算 | `import numpy as np; np.load("weights.npy")` |
| **scipy** | 1.17.1 | 科学计算/统计 | `scipy.stats.norm.fit(data)` |
| **Pillow** | 11.3.0 | 图像处理（对抗样本可视化） | `from PIL import Image; Image.open("adv.png")` |
| **scikit-learn** | 1.8.0 | 传统 ML、评估与辅助攻击建模 | `from sklearn.metrics import roc_auc_score` |

### 需安装的 Python 包（不在 ctf-tools 中）

```bash
conda activate ctf-tools
pip install torch transformers safetensors
conda deactivate
```

### 系统组件（WSL Kali）

| 工具 | 路径 | 用途 |
|---|---|---|
| **python3-dev** | WSL 系统已安装 | C 扩展编译头文件 |
