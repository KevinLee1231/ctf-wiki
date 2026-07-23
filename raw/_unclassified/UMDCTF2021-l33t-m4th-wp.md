# l33t m4th

## 题目简述

题目连续给出使用 leetspeak 和文字描述写成的算术表达式，要求解析变量、运算符和数字并返回结果。完成全部轮次后，服务端输出 flag。

## 解题过程

先把稳定的 leetspeak 词汇归一化，例如把 `4`、`3`、`1`、`0` 在对应单词中的替代关系还原，再识别：

- 数字常量；
- 变量赋值；
- 加、减、乘、除和括号；
- 后续题目对已有变量的引用。

不要直接对未经约束的远端文本调用 Python `eval()`。可以先词法化，再用受限 AST 只允许算术节点：

```python
import ast
import operator

OPS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.FloorDiv: operator.floordiv,
}

def evaluate(node, names):
    if isinstance(node, ast.Constant) and isinstance(node.value, int):
        return node.value
    if isinstance(node, ast.Name):
        return names[node.id]
    if isinstance(node, ast.BinOp) and type(node.op) in OPS:
        return OPS[type(node.op)](
            evaluate(node.left, names),
            evaluate(node.right, names),
        )
    raise ValueError("unsupported expression")
```

按协议保存赋值并逐轮回答，最终得到：

```text
UMDCTF-{h4h4_w311_4t_134$7_th4ts_f1n4lly_0v3r_r1gh7?}
```

## 方法总结

这题属于文本解析与状态维护。可靠解法应把“leetspeak 归一化、词法/语法解析、变量环境、网络交互”分层处理；受限 AST 既比正则拼凑稳健，也避免了直接 `eval` 远端输入的代码执行风险。
