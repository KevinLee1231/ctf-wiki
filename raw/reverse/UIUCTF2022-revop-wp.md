# revop

## 题目简述

附件包含残缺的 `model_dist.onnx` 和由参考模型编译出的 x86-64 ELF 共享库 `model_src.so`。ONNX 图保留了权重和拓扑，但其中三个节点只有名字 `customop_1`、`customop_2`、`customop_3`，没有标准实现。目标是从共享库恢复三个算子的逐元素语义，把它们替换为标准 ONNX 节点，再提交完整模型。

服务对一次随机生成的输入 `torch.randn(64, 1, 32, 32)` 分别运行提交模型和隐藏的 `model_ref.onnx`，并使用：

```python
np.testing.assert_allclose(candidate, reference, rtol=1e-3, atol=1e-5)
```

比较输出。因此只伪造 shape 或常量输出无法稳定通过，必须让修复后的推理图与参考实现数值等价。虽然载体是 ML 模型，决定性障碍是恢复编译后的自定义运行时逻辑，所以归入 Reverse。

## 解题过程

### 在共享库中定位三个逐元素算子

`model_src.so` 带 debug info 且未 strip，但模型被合并进单个 `main_graph`。沿 ONNX 图中最后一组 Gemm 后的三个未知节点，在 `main_graph` 尾部可观察到三个连续、长度相同的张量循环：

1. `0x45fd` 调用 `cosf@plt`，`0x46e9` 调用 `sinf@plt`，随后 `0x481f` 用 `addss` 合并同一位置的两个结果；
2. 下一段先计算 Tanh，`0x4b9d` 用 `subss` 从输入中减去该结果；
3. `0x4c93` 读取 `.rodata` 地址 `0x7000` 的浮点常量，原始字节为 `0a d7 23 3c`，按小端解释是 `0.01f`；后续对负数分支乘该常量并与非负分支选择合并。

因此三个公式分别为：

$$
f_1(x)=\cos x+\sin x,
$$

$$
f_2(x)=x-\tanh x,
$$

$$
f_3(x)=\begin{cases}
x,&x\ge0,\\
0.01x,&x<0.
\end{cases}
$$

第三个算子就是 `alpha=0.01` 的 LeakyReLU；该值也是 ONNX `LeakyRelu` 的默认 alpha，但仍应从二进制常量核对，不能只凭默认值猜测。

### 用标准节点替换自定义链

使用 ONNX GraphSurgeon 读取图，复用 `customop_1` 的输入和 `customop_3` 的最终输出，在中间创建新变量：

```python
import numpy as np
import onnx
import onnx_graphsurgeon as gs

graph = gs.import_onnx(onnx.load("model_dist.onnx"))

op1 = next(node for node in graph.nodes if node.op == "customop_1")
op2 = next(node for node in graph.nodes if node.op == "customop_2")
op3 = next(node for node in graph.nodes if node.op == "customop_3")

x = op1.inputs[0]
final_outputs = list(op3.outputs)

cos_v = gs.Variable("revop_cos", dtype=np.float32)
sin_v = gs.Variable("revop_sin", dtype=np.float32)
add_v = gs.Variable("revop_add", dtype=np.float32)
tanh_v = gs.Variable("revop_tanh", dtype=np.float32)
sub_v = gs.Variable("revop_sub", dtype=np.float32)

replacement = [
    gs.Node(op="Cos", inputs=[x], outputs=[cos_v]),
    gs.Node(op="Sin", inputs=[x], outputs=[sin_v]),
    gs.Node(op="Add", inputs=[cos_v, sin_v], outputs=[add_v]),
    gs.Node(op="Tanh", inputs=[add_v], outputs=[tanh_v]),
    gs.Node(op="Sub", inputs=[add_v, tanh_v], outputs=[sub_v]),
    gs.Node(
        op="LeakyRelu",
        attrs={"alpha": 0.01},
        inputs=[sub_v],
        outputs=final_outputs,
    ),
]

# 让 cleanup 删除原来的三个死节点；最终输出已由 LeakyRelu 接管。
op1.outputs.clear()
op2.outputs.clear()
op3.outputs.clear()
graph.nodes.extend(replacement)

graph.cleanup().toposort()
onnx.checker.check_model(gs.export_onnx(graph))
onnx.save(gs.export_onnx(graph), "model_solved.onnx")
```

保存前显式设置 `alpha` 可避免不同工具链对默认属性的处理差异。还应在本地用 ONNX Runtime 检查修复模型能够加载、输入输出名称和 shape 未改变；若持有参考模型，再对多组随机输入执行与服务相同的 `assert_allclose`。

### 提交模型

服务接受 Base64 编码的 ONNX 文件：

```python
import base64
from pathlib import Path

payload = base64.b64encode(Path("model_solved.onnx").read_bytes())
io.sendlineafter(b"Input base64 onnx file:", payload)
print(io.recvuntil(b"}").decode())
```

修复后的图通过标准 ONNX checker，并在随机输入上与隐藏参考模型保持容差内一致，得到：

```text
uiuctf{c00l_@nd_cr8t1v3_mL_0p5_fr0m_pwnY_0p5_!!!!}
```

## 方法总结

- 核心技巧：把编译模型尾部的逐元素循环与 ONNX 缺失节点一一对齐，恢复 `cos+sin`、`x-tanh(x)` 和 `alpha=0.01` 的 LeakyReLU，再用标准算子重建图。
- 识别信号：分发模型权重和整体拓扑完整，仅有少量自定义 op；参考共享库中又出现对应数学库调用、固定张量循环和浮点常量。
- 复用要点：恢复自定义算子时要同时核对运算顺序、输入来源、广播/shape、dtype 和激活参数。仅凭符号名猜函数不够，最终必须让 ONNX checker、运行时加载和多组随机输入的数值比较都通过。
