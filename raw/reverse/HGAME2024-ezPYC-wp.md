# ezPYC

## 题目简述

附件是由 PyInstaller 打包的 Python 3.8 可执行文件。解包后可以取得入口字节码 `ezPYC.pyc`，反编译结果显示程序用循环密钥 `[1, 2, 3, 4]` 对 flag 字节逐位异或。

## 解题过程

先用 `pyinstxtractor.py` 提取 PyInstaller 归档：

```powershell
python.exe ".\pyinstxtractor.py" ".\ezPYC.exe"
```

工具会创建 `ezPYC.exe_extracted` 目录。官方输出同时提示打包程序使用 Python 3.8；如果解释器版本不一致，解包阶段可能无法正确反序列化字节码，因此应尽量使用对应版本运行提取工具。

在提取目录找到 `ezPYC.pyc` 后，使用 `pycdc` 反编译：

```powershell
pycdc.exe ".\ezPYC.exe_extracted\ezPYC.pyc"
```

反编译得到的关键数据和解密逻辑可以整理成：

```python
ciphertext = bytes(
    [
        87, 75, 71, 69, 83, 121, 83, 125, 117, 106, 108, 106,
        94, 80, 48, 114, 100, 112, 112, 55, 94, 51, 112, 91,
        48, 108, 119, 97, 115, 49, 112, 112, 48, 108, 100, 37,
        124, 2,
    ]
)
key = bytes([1, 2, 3, 4])

plaintext = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(ciphertext)
).rstrip(b"\x00")
print(plaintext.decode())
```

输出为：

```text
VIDAR{Python_R3vers3_1s_1nter3st1ng!}
```

## 方法总结

- 核心技巧：识别 PyInstaller、提取 `.pyc`、反编译字节码，再还原循环 XOR。
- 识别信号：可执行文件包含 `pyi_`/`pyiboot` 字符串，提取器能识别 PyInstaller 与内嵌 Python 版本。
- 复用要点：解包解释器版本应与打包版本匹配；反编译失败时仍可用 `dis` 或字节码工具检查常量和循环结构，不应把解包成功等同于已经完成逆向。
