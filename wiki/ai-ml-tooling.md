---
type: tooling
tags: [ai-ml, tooling, tools, environment]
skills: [ctf-ai-ml]
---

# AI/ML Tooling

本页维护模型、权重、推理接口和 LLM/Agent 题目的本机工具入口。普通 HTTP 漏洞转 [web-tooling.md](web-tooling.md)；只有模型行为、训练数据或工具决策决定解法时才使用本页。

## 平台与当前工具

AI/ML 分析默认使用 Windows venv：`D:/文档/新建文件夹/venv/Scripts/python.exe`（Python 3.14.3）。WSL `ctf-tools` 不承担模型分析，也没有通用数值、图像或深度学习工具合集。

| Windows venv 包 | 当前版本 | 用途与边界 |
|---|---|---|
| `torch` | `2.12.0+cpu` | CPU 张量计算、自动求导及 PyTorch 模型推理；当前 `cuda.is_available()` 为 false |
| `onnxruntime` | `1.26.0` | ONNX 推理；当前 provider 列表含 CPU 与 Azure，没有 CUDA provider；本地推理显式选 CPU |
| `numpy` / `scipy` | `2.4.4` / `1.17.1` | 数组、优化、距离、频谱与数值实验 |
| `pandas` / `scikit-learn` | `3.0.2` / `1.9.0` | 样本整理、传统分类模型与统计基线 |
| `lightgbm` | `4.6.0` | 读取、运行和分析已有树模型 |
| `Pillow` / `matplotlib` | `12.2.0` / `3.11.0` | 输入图像、扰动检查、曲线和热图 |
| `requests` | `2.33.1` | 有界调用题目推理 API，记录输入、输出及查询预算 |

这些是本机已安装包；模型文件、权重和 tokenizer 不随包自动就绪。`transformers`、`safetensors` 当前未在此 venv 中发现，需要时先评估 Windows 兼容版本。不得照旧清单向 WSL 安装 PyTorch、CUDA 或 Triton。

## 从 pwsh 调用

先确认计算设备，再执行题目脚本：

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.tensor([1.,2.]).sum())'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import onnxruntime as ort; print(ort.get_available_providers())'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze_model.py"
```

在该解释器运行的脚本中，本地 ONNX 推理可显式固定执行后端：

```python
import onnxruntime as ort

session = ort.InferenceSession(
    "D:/题目路径/model.onnx",
    providers=["CPUExecutionProvider"],
)
for item in session.get_inputs():
    print(item.name, item.shape, item.type)
```

PyTorch checkpoint 优先使用 `torch.load(path, map_location="cpu", weights_only=True)` 读取权重；自定义对象报错时先核对题目源码与序列化格式，不把关闭限制作为自动处理步骤。输入 shape、dtype、归一化和类别顺序必须与目标模型一致；“能加载”不等于复现了题目推理。

LLM/Agent 题先记录消息层级、实际工具接口、输出判定和请求预算。可用的浏览器、搜索或 MCP 由当前会话发现，不从历史工具表猜测接口；基础路线见 [llm-attacks.md](llm-attacks.md)。

## 失败处理

- `CUDA unavailable` 或没有 CUDA provider：当前环境就是 CPU 路线，先用小样本验证。确需加速时单独评估 Windows 依赖、硬件和资源预算，不自动改用 WSL。
- 权重格式无法加载：确认 framework、文件内容与模型定义，转 [ml-model-inference-extraction-and-weight-analysis.md](ml-model-inference-extraction-and-weight-analysis.md)。缺包先评估 Windows 新版本或格式转换所需的最小工具。
- 梯度、概率或攻击效果不符：先检查预处理、eval/train 状态、随机种子、损失函数和 query 计数，再增加优化规模。
- 只有鉴权、反代、请求字段或服务端漏洞：转 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)。
- 任何 WSL 新增、升级或重装，都必须先说明为何任务依赖 Linux、安装目标和主要依赖影响，并取得明确同意；普通“跑通模型”不包含这项授权。
