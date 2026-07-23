# UMDCTF2022 Legacy Writeup

## 题目简述

服务端明确使用 Python 2.7，并随机生成一个几乎不可能在三次机会内猜中的 `secret`。真正的问题不在随机数强度，而在 Python 2 的 `input()`：它会把用户输入当成 Python 表达式求值。

决定性障碍是语言运行时造成的执行边界突破，因此归入 `pwn`。

## 解题过程

核心代码如下：

```python
secret = random.randrange(0, 1000000000000000514)

for i in range(3):
    if input(str(3-i) + " chances left! \n") == secret:
        print(flag)
```

Python 3 的 `input()` 返回字符串，而 Python 2 的 `input()` 近似于：

```python
eval(raw_input())
```

表达式求值发生在当前全局作用域，所以无需猜测随机数，只要输入变量名本身：

```text
secret
```

服务端会先把它求值为全局变量 `secret` 的当前值，再执行：

```python
secret == secret
```

比较恒为真，直接输出：

```text
UMDCTF{W3_H8_p7th0n2}
```

## 方法总结

本题利用的是 Python 2 与 Python 3 同名 API 的语义差异。遇到旧运行时服务时，不能只按现代语言习惯理解输入函数；应先确认输入是普通文本、字面量解析，还是任意表达式求值。这里甚至不需要调用函数或执行系统命令，读取当前作用域里的变量就足以绕过认证。
