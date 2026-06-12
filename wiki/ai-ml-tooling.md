---
type: tooling
tags: [ai-ml, tooling, tools, environment]
skills: [ctf-ai-ml]
updated: 2026-05-21
---

# AI/ML Tooling

本页记录 `ctf-ai-ml` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 调用层与覆盖状态

### 非交互调用原则

- 首轮只依赖 `ctf-tools` 中已经可用的 `numpy`、`scipy`、`Pillow`、`scikit-learn`，避免为不确定题型先安装重量级框架。
- 需要深度学习框架时，先确认题目确实给出模型权重、推理接口、adapter、embedding 或训练数据，再按需补装。
- 如果只是 Web 上的模型服务、prompt/API 权限问题，先回到 `ctf-web` 或对应服务链路，不要直接装 ML 框架。

### 知识页覆盖状态

- 当前覆盖 adversarial input、LLM/prompt、model extraction/inversion/LoRA 等核心方向。
- 覆盖仍偏轻；后续 raw 若出现 ONNX、PyTorch checkpoint、safetensors、TensorFlow Lite、Kaggle 风格数据泄漏，应优先沉淀为独立技巧页。

### 后续补强方向

- 模型文件格式 triage：`.pt/.pth/.onnx/.safetensors/.tflite`。
- 权重/adapter 差分：LoRA merge、恶意 adapter、embedding 泄露。
- 远程推理 oracle：query budget、置信度侧信道、rate limit 绕过。

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
