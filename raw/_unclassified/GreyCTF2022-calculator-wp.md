# GreyCTF2022 - Calculator

## 题目简述

服务连续给出 100 道前缀表达式，要求在 30 秒内作答。核心是实现稳定的前缀表达式解析器并自动交互，不涉及漏洞或密码学。

## 解题过程

前缀表达式可从右向左扫描：数字入栈，遇到运算符则依次弹出左、右操作数，计算后把结果压回栈。由于扫描方向反转，弹出顺序不能写反。

```python
def evaluate(tokens):
    stack = []
    for token in reversed(tokens):
        if token.lstrip('-').isdigit():
            stack.append(int(token))
        else:
            left = stack.pop()
            right = stack.pop()
            stack.append(apply(token, left, right))
    assert len(stack) == 1
    return stack[0]
```

用交互脚本逐行提取表达式、调用求值器并发送答案。除法必须遵循服务语言的整数截断规则，负数也要按一个 token 处理。完成 100 轮后得到：

```text
grey{prefix_operation_is_easy_to_evaluate_right_W2MQshAYVpGVJPcw}
```

## 方法总结

自动答题首先要分离协议解析与算法求值，并为负数、除零、整数除法和多位数字编写本地测试。前缀表达式不需要括号；右到左栈算法复杂度为 $O(n)$，足以满足严格时限。
