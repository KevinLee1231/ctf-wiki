# Welcome to Python

## 题目简述

题目给出一个由 PyInstaller 打包的 Linux 单文件可执行程序。恢复 Python 字节码后，可以看到一个逐字符校验器：它用三角函数和浮点位模式生成掩码，再与输入字符组合并和常量数组比较。

## 解题过程

二进制中存在 PyInstaller 相关字符串和归档结构。使用 `pyinstxtractor` 解包后，找到
`chal.pyc`；题目构建环境是 Python 3.10，因此应使用兼容该字节码版本的
`pycdc`、`decompyle3` 或直接反汇编 `marshal` 代码对象恢复逻辑。

每个位置先计算：

```python
def wandom(x):
    return x * x * cos(x) * sin(x) / 1000

def evil_bit_hack(y):
    return int(c_uint32.from_buffer(c_float(y)).value)
```

其中 `evil_bit_hack` 不是数值取整，而是把 32 位浮点的原始 IEEE-754 位模式解释为无符号整数。校验式为：

```python
c = ~(~ord(ch) ^ evil_bit_hack(wandom(wandom(w))) & evil_bit_hack(w)) + 1
```

按 Python 位运算优先级反解，官方脚本对每个常量执行：

```python
ch = chr(
    ~(
        evil_bit_hack(w)
        & evil_bit_hack(wandom(wandom(w)))
        ^ (~source[i] + 1)
    )
)
```

循环索引从 `seed = 64` 开始，必须使用与题目一致的单精度 `c_float` 位模式；直接使用 Python 双精度数的整数表示会产生不同掩码。

逐项恢复并连接得到：

```text
UMDCTF{0_0+-+eXP-eLLiARm_us_!!!-12345}
```

## 方法总结

- 核心技巧：先识别并拆出 PyInstaller 内嵌归档，再对真正的 Python 校验器做代数逆运算。
- `c_float` 到 `c_uint32` 是位级重解释，不是普通类型转换。
- 解包工具和反编译器必须匹配 Python 3.10 字节码；版本错配时可退回 `dis` 查看指令。
- 校验长度与常量数组长度一致，恢复时应遍历全部 38 项，不能只利用“部分正确”提示截断。
