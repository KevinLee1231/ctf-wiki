# Discrepancy

## 题目简述

题目要求提交五个不超过 8 字节的 Python Pickle 数据，使 CPython C 版 `_pickle.Unpickler`、纯 Python `pickle._Unpickler` 和 `pickletools.dis` 对同一字节串产生不同的接受/拒绝结果。

决定性障碍是理解三套 Pickle 实现的 opcode 与解析边界差异，因此从仓库的 `misc` 调整到 `reverse`。

## 解题过程

### 1. `(\x88.`：反汇编器拒绝多余栈元素

```python
b"(\x88."
```

含义是：

```text
MARK
NEWTRUE
STOP
```

两个 Unpickler 在 `STOP` 时返回栈顶 `True`，不要求栈中只剩一个对象；`pickletools.dis` 则检查终止时的抽象栈，发现 `MARK` 仍然残留而拒绝。

### 2. `\x88(e.`：空 `APPENDS` 的实现差异

```python
b"\x88(e."
```

含义是：

```text
NEWTRUE
MARK
APPENDS
STOP
```

MARK 后没有待追加元素。C 快速路径把空追加视为无操作，`pickletools` 的静态栈模型也接受；纯 Python 实现仍尝试取得目标的 `append`/`extend` 行为，而目标是布尔值，因而报错。

### 3. `F 5\n.`：FLOAT 前导空白

```python
b"F 5\n."
```

`FLOAT` 的文本参数是 `" 5"`。纯 Python 使用 `float()`，允许前导空白；`pickletools` 也按数值语法接受。C 版路径采用更严格的文本格式检查，拒绝该表示。

### 4. `(.`：MARK 被当作可返回值

实际最小字节串为：

```python
b"(."
```

`pickletools` 的抽象解释把 MARK 当作栈元素，随后允许 `STOP` 返回它；两个真实 Unpickler 则把 MARK 视为内部哨兵，不允许把它反序列化为 Python 对象，因此拒绝。

### 5. `I1\x00\n.`：NUL 后缀整数

```python
b"I1\x00\n."
```

文本 INT 参数含嵌入 NUL。C 实现底层整数转换在 NUL 处停止，读到有效前缀 `1` 并接受；纯 Python 的 `int()` 与 `pickletools` 会看到完整字符串中的非法 NUL，因而拒绝。

### 6. 最终五组

按上述顺序提交：

```python
payloads = [
    b"(\x88.",
    b"\x88(e.",
    b"F 5\n.",
    b"(.",
    b"I1\x00\n.",
]
```

每组均不超过 8 字节，并分别命中题目要求的接受矩阵。

仓库正式挑战目录中的 `flag.txt` 给出：

```text
SEKAI{p1ckleeeeeeeee_3a01fea10fb01a88c1cd554e7372f21ced43b497}
```

## 方法总结

这道题展示了“格式规范”“静态反汇编器”和“两个运行时实现”并不天然等价。差异主要来自：

- `STOP` 时是否检查完整栈形状；
- 空批量操作是否走快速路径；
- 文本数字是否接受空白或 NUL 后缀；
- 内部哨兵是否可能成为结果。

跨语言或跨实现验证不应假设一个解析器的接受集合等于另一个。安全协议若先用工具 A 检查、再用运行时 B 执行，就必须针对这种解析差异做差分测试。
