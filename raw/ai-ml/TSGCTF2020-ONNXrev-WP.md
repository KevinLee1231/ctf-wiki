# TSGCTF2020 ONNXrev WP

## 题目简述

程序要求输入 41 字符，并使用 40 像素的 Inconsolata Regular 等宽字体把字符串渲染成大小为 $42\times820\times3$ 的 RGB 图像，再交给 `problem.onnx` 判断。模型并不是普通的字符分类器：它先对每个 $20\times42$ 字符块计算一个整数特征，再用隐藏在 ONNX 常量节点中的循环卷积系数检验 41 个线性等式。

复现时必须使用题目指定版本的 [Inconsolata-Regular.ttf](https://github.com/googlefonts/Inconsolata/blob/e0c6cfb8df929029c123fa01d036a81b3146d0e7/fonts/ttf/Inconsolata-Regular.ttf)，其 SHA-1 为 `b29224e69806e6ffe412082795ac0fc8cc312951`；字体版本变化会改变像素和模型输出。

## 解题过程

先用 ONNX 可视化器或直接遍历 `model.graph.node`，可还原出模型的主体逻辑。设字符串长度 $N=41$，第 $j$ 个字符图像经过内部子图 `oracle` 后得到整数 $x_j$；模型对每个 $i$ 检查：

$$
\sum_{j=0}^{N-1} A_{i,(i-j)\bmod N}x_j=B_i.
$$

`A` 和 `B` 分别位于顶层图第 4、3 个节点的 tensor 属性中。官方脚本按模型结构取出它们，并把深层 `If/Loop` 属性里真正负责字符特征的子图单独保存成可执行 ONNX 模型：

```python
model = onnx.load("problem.onnx")
A = model.graph.node[3].attribute[0].t.int64_data
B = model.graph.node[2].attribute[0].t.int64_data
N = len(B)

oracle_graph = model.graph.node[4].attribute[0].g.node[7].attribute[0].g
onnx.save(helper.make_model(oracle_graph), "oracle.onnx")
```

接着把 $x_0,\ldots,x_{40}$ 视为整数未知量，用 Z3 建立 41 个线性方程。这里系数索引必须按循环卷积还原，不能直接写成普通矩阵的 `A[i][j]`：

```python
x = [Int(f"x{i}") for i in range(N)]
s = Solver()
for i in range(N):
    expr = 0
    for j in range(N):
        ij = (i - j + N) % N
        expr += A[i * N + ij] * x[j]
    s.add(expr == B[i])
assert s.check() == sat
values = [s.model()[v].as_long() for v in x]
```

求出的只是每个字符图块应产生的 oracle 值，还需建立“特征值到字符”的查找表。对每个可打印字符 `c`，渲染 `c + "g" * 40`；选择 `g` 是为了让整行字体包围盒稳定在 `(820, 42)`，而分号 `;` 的高度为 43，必须从候选集中排除。运行拆出的子图时把循环计数设为 0、系数设为全 1，并取第一个字符对应的输出：

```python
letters = "".join(c for c in string.printable
                  if c not in string.whitespace + ";")
table = {}
for c in letters:
    image = render(c + "g" * (N - 1))
    result = session.run(None, oracle_inputs(image))
    table[result[3][0]] = c

flag = "".join(table[v] for v in values)
```

最终恢复：

```text
TSGCTF{OnNx_1s_4_kiNd_0f_e5oL4ng_I_7hink}
```

## 方法总结

本题的决定性障碍是恢复 ONNX 计算图中的模型行为，因此归入 AI/ML。解法不是训练替代模型，而是把网络拆成“字符渲染、单字符整数 oracle、循环线性约束”三层：先从常量节点提取 $A,B$ 并解线性系统，再对有限字符集运行原始子图建立反查表。处理以图像为输入的模型逆向时，字体文件、字号、包围盒和预处理维度都是算法的一部分，任何一项不一致都会使同一字符产生不同特征。
