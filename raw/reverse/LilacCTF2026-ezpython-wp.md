# ezPython

## 题目简述

题目是 PyInstaller 打包的 Python 逆向。`main.py` 中调用了 `a85decode`，但没有写成 `base64.a85decode`。关键点在 PyInstaller runtime hook：程序启动时把自定义的 `a85decode` 和 `b64decode` 塞进了 `builtins`，因此主程序实际调用的不是标准库函数，而是被 hook 过的函数。

官方反编译片段显示：

```python
import builtins
from stdlib import a85decode, b64decode
builtins.a85decode = a85decode
builtins.b64decode = b64decode
```

## 解题过程

### 关键观察

自定义 `a85decode` 表面上仍在做 Ascii85 解码，但中途会篡改 `struct.__code__`，把若干字节码中的位移方向和常量改掉。被修改的目标是 `MX` 函数，也就是 XXTEA/BTEA 的轮函数。

原始反编译中看到的 `MX` 类似：

```python
def MX(y, z, sum, k, p, e):
    return ((z >> 5) ^ (y >> 2)) + ((y << 3) ^ (z << 4)) ^ (sum ^ y) + (k[(p & 3) ^ e] ^ z)
```

但经过 hook 后，真实参与运算的是被修正过的版本：

```python
def true_MX(y, z, sum, k, p, e):
    return ((z << 3) ^ (y >> 5)) + ((y << 4) ^ (z >> 2)) ^ (sum ^ y) + (k[(p & 3) ^ e] ^ z)
```

因此直接照反编译代码解密会失败，必须先还原 runtime hook 对字节码造成的 SMC 效果。

### 求解步骤

恢复真实 `MX` 后，按 XXTEA 解密流程处理常量密文：

```python
res = [761104570, 1033127419, 3729026053, 795718415]
key = struct.unpack("<IIII", b"1111222233334444")
btea(res, -4, key)
plain = struct.pack("<IIII", *res)
```

用修正后的 `true_MX` 解密可得到：

```text
e@sy_Pyth0n_SMC!
```

这说明题目的核心不是 XXTEA 本身，而是 PyInstaller runtime hook 对内置函数和字节码的动态篡改。

## 方法总结

- 核心技巧：检查 PyInstaller runtime hook，确认主程序调用的 builtin 是否被替换，再恢复被 SMC 修改后的真实字节码逻辑。
- 识别信号：源码里直接调用未限定命名空间的函数，但标准 import 中看不到定义，应检查 `builtins`、runtime hook 和打包入口。
- 复用要点：Python 逆向不能只信静态反编译结果；hook 修改 `__code__`、替换 builtin、动态导入都会让反编译逻辑与真实执行逻辑不一致。
