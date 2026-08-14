# Basic Calculator

## 题目简述

附件是 PyInstaller 打包的 Windows 计算器。普通输入只执行 `eval(expression)`；当结果等于 `1337`，并且原表达式同时包含 `*` 和字符串 `191` 时，程序调用隐藏的 `magic(1337)`。该函数把一组整数逐项与 1337 异或，结果即为 Flag。

## 解题过程

先用 `pyinstxtractor` 从 EXE 中提取 PYZ 和 `.pyc`，再使用与目标 Python 版本匹配的反编译器恢复主脚本。关键分支为：

```python
result = str(eval(expression))
if result == "1337":
    if "*" in expression and "191" in expression:
        input_text.set(magic(int(result)))
```

满足条件的最简单表达式是：

```text
191*7
```

也可以直接在恢复脚本中执行核心函数：

```python
arr = [
    1374, 1355, 1372, 1344, 1361, 1368, 1357,
    1354, 1346, 1353, 1344, 1370, 1382, 1387,
    1290, 1382, 1288, 1354, 1382, 1353, 1401,
    1360, 1367, 1407, 1388, 1365, 1304, 1348,
]
print("".join(chr(x ^ 1337) for x in arr))
```

输出为：

```text
greyhats{pyc_R3_1s_p@inFUl!}
```

## 方法总结

- 核心技巧：从 PyInstaller 可执行文件恢复 Python 字节码，定位隐藏 GUI 分支并逆转 XOR。
- 识别信号：体积较大的 Python GUI EXE、PyInstaller 特征、界面功能与内部存在大量无关辅助函数。
- 复用要点：反编译结果不完整时优先恢复常量、控制流和关键函数，不必修复整个 GUI 程序。
