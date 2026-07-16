# 二进制学徒

## 题目简述

附件是一个 Python 字节码文件。程序运行后只会输出“我的 flag 去哪儿了？”，因此关键不是执行结果，而是检查 `.pyc` 中保留的代码对象与常量。

## 解题过程

文件头魔数为 `A7 0D 0D 0A`，对应 Python 3.11 字节码。使用同版本 Python 跳过 16 字节文件头，再通过标准库 `marshal` 读取顶层代码对象：

```python
import marshal

with open("二进制学徒.pyc", "rb") as f:
    f.read(16)
    code = marshal.load(f)

for value in code.co_consts:
    if isinstance(value, str):
        print(repr(value))
```

输出中可以看到一个没有参与主流程的字符串常量：

```text
'\n0xGame{15cf69f6-69f9-6b18-1464-7785f106b0d3}\n'
```

反编译结果同样表明，该字符串位于模块顶层，而 `main` 函数只负责执行 `print("我的flag去哪儿了？")`。

因此 flag 为：

```text
0xGame{15cf69f6-69f9-6b18-1464-7785f106b0d3}
```

## 方法总结

`.pyc` 并不会隐藏源码中的常量。遇到这类题目，应先根据魔数选择匹配的 Python 版本，再检查顶层及嵌套代码对象的 `co_consts`；即使某个字符串不在可达执行路径中，也可能完整保留在字节码里。
