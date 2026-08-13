# GreyCTF2022 - oneliner

## 题目简述

附件是一条接近 10 MB 的嵌套条件表达式，用大量三元运算隐藏逐字符检查。程序体积很大，但每个条件最终只约束某个固定下标的字符，没有跨字符复杂状态。

## 解题过程

不要手工格式化整个文件。先用语言解析器构建 AST，递归遍历条件节点，提取形如 `input[index]` 与常量之间的比较；若表达式对字符做了加减或异或，则按相反运算恢复目标字节。

```python
constraints = {}

def visit(node):
    if is_char_check(node):
        index, relation = parse_char_check(node)
        constraints[index] = invert_relation(relation)
    for child in children(node):
        visit(child)

visit(parse(source))
answer = bytes(constraints[i] for i in range(max(constraints) + 1))
```

另一种做法是把输入字符建模为位向量，只保留能到达成功分支的条件交给求解器。恢复并运行原检查器确认后得到：

```text
grey{quite_big_ah}
```

## 方法总结

代码体积不是算法复杂度。面对机器生成的一行式，应先识别语法树中重复的局部模板，提取约束而不是美化全部源码。最终要把恢复字符串送回原程序验证成功分支，防止三元分支方向或运算优先级解析错误。
