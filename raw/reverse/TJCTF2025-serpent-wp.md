# serpent

## 题目简述

附件 `ast_dump.pickle` 是序列化后的 Python 抽象语法树，题目提示“go to the deepest level”。flag 被放在深层嵌套节点中的赋值语句 `secret = <常量>` 里；无需把整棵 AST 重新生成源码，只需递归遍历子节点并识别目标赋值。

## 解题过程

Python 的 `ast.iter_child_nodes` 能统一遍历函数、类、条件、表达式等不同节点的子树。对每个 `ast.Assign` 检查左侧是否包含名为 `secret` 的 `ast.Name`，并确认右侧是 `ast.Constant`；同时记录递归深度，最后取最深的候选。

`pickle.load` 可能执行序列化数据中携带的任意构造逻辑，因此只应在隔离环境中处理可信的比赛附件；对未知来源文件，可先用 `python -m pickletools ast_dump.pickle` 做静态检查。

```python
import ast
import pickle

def find_secrets(node, depth=0):
    if isinstance(node, ast.Assign) and isinstance(node.value, ast.Constant):
        for target in node.targets:
            if isinstance(target, ast.Name) and target.id == "secret":
                yield depth, node.value.value

    for child in ast.iter_child_nodes(node):
        yield from find_secrets(child, depth + 1)

with open("ast_dump.pickle", "rb") as f:
    tree = pickle.load(f)

depth, flag = max(find_secrets(tree), key=lambda item: item[0])
print("depth:", depth)
print(flag)
```

在发布附件中，目标赋值位于递归深度 16，输出为：

```text
tjctf{f0ggy_d4ys}
```

当前 Python 版本加载该附件时可能提示若干 AST 构造器缺少字段的弃用警告，这是旧版序列化 AST 与新版构造器校验差异造成的；不影响本题读取已有节点和值，但未来 Python 版本可能把它升级为错误。

## 方法总结

- 核心技巧：按 AST 节点类型递归遍历，并以变量名、赋值类型和深度共同定位隐藏常量。
- 识别信号：附件是 pickled AST、题面强调嵌套深度、目标变量名具有明确语义。
- 复用要点：不要用针对某一种语法结构的硬编码字段链；`ast.iter_child_nodes` 更能覆盖异构节点，同时必须把反序列化风险与题目逻辑分开处理。
