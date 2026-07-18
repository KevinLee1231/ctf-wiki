# week4exe

## 题目简述

附件 `1.exe` 是一个 64 位 PyInstaller 程序，内嵌 Python 3.6 字节码。核心逻辑先对用户输入做 Base64 编码，再把相邻两个编码字节异或，最后与固定数组比较。解题重点是正确提取入口模块并从字节码还原算法，而不是盲目修补 `.pyc` 文件头。

## 解题过程

检查 PE 尾部可以确认文件带有 PyInstaller overlay。使用支持自动识别 Python 版本的 `pyinstxtractor-ng` 解包：

```text
pyinstxtractor-ng -d 1.exe
```

实际入口模块是 `easypy.pyc`。提取出的文件已经具有 Python 3.6 所需的 12 字节 `.pyc` 头，字节码对象从偏移 `0x0c` 的 `E3` 开始；原稿中“从其他文件复制 16 字节文件头”的做法会把 Python 3.7 之后的头部长度误套到 Python 3.6 上，既不必要也可能破坏文件。

若当前版本的 `uncompyle6` 无法反编译 Python 3.6 字节码，可以直接反汇编：

```text
pydisasm -F extended easypy.pyc
```

由字节码可还原出固定数组和变换逻辑：

```python
check = [
    5, 32, 6, 55, 14, 102, 93, 9, 86, 87, 8, 33,
    26, 26, 58, 21, 53, 1, 48, 2, 33, 124, 95, 50,
    103, 84, 82, 108, 14, 103, 74, 28, 55, 97, 13, 41
]

encoded = base64.b64encode(user_input)
result = [encoded[i] ^ encoded[i + 1] for i in range(len(encoded) - 1)]
result.append(encoded[-1] ^ 20)
```

设 Base64 编码结果为 $e_i$、比较数组为 $c_i$，则最后一个字节满足 $e_{n-1}=c_{n-1}\oplus20$，其余字节满足 $e_i=c_i\oplus e_{i+1}$。从后向前递推即可恢复完整 Base64 字符串：

```python
import base64

check = [
    5, 32, 6, 55, 14, 102, 93, 9, 86, 87, 8, 33,
    26, 26, 58, 21, 53, 1, 48, 2, 33, 124, 95, 50,
    103, 84, 82, 108, 14, 103, 74, 28, 55, 97, 13, 41
]

encoded = [0] * len(check)
encoded[-1] = check[-1] ^ 20
for i in range(len(check) - 2, -1, -1):
    encoded[i] = check[i] ^ encoded[i + 1]

encoded = bytes(encoded)
plain = base64.b64decode(encoded)
print(encoded.decode())
print(plain.rstrip(b"\r\n").decode())
```

输出为：

```text
MHhnYW1le3dlMWMwbWVfdE9fT3g5YW0zfQ0=
0xgame{we1c0me_tO_Ox9am3}
```

解码结果末尾原本带有 `\r`，这是 Windows 输入换行残留，不属于 flag。

## 方法总结

本题的可靠流程是“识别 PyInstaller → 提取实际入口 `easypy.pyc` → 按 Python 3.6 格式读取或反汇编 → 逆推相邻异或”。处理 `.pyc` 时应先根据 Python 版本判断头部结构；反编译器不兼容并不妨碍直接从字节码恢复短小算法。
