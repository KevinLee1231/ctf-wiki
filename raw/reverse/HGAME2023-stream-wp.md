# stream

## 题目简述

附件是由 PyInstaller 打包的 Windows Python 可执行文件。解包并反编译入口字节码后，可以看到程序使用固定密钥生成 RC4 密钥流，将输入逐字节异或，再进行 Base64 编码并与常量字符串比较。

## 解题过程

PyInstaller 程序通常包含 `PYZ`、`pyi_` 引导模块以及内嵌 Python DLL。使用 [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) 解包：

```bash
python pyinstxtractor.py stream.exe
```

输出目录中可以找到入口文件 `stream.pyc`。用与样本 Python 版本兼容的反编译器恢复代码后，得到密钥与密文：

```python
key = "As_we_do_as_you_know"
enc = (
    "wr3ClVcSw7nCmMOcHcKgacOtMkvDjxZ6asKWw4nChMK8IsK7KMOOasOrdgbDlx3Dqc"
    "Kqwr0hw701Ly57w63CtcOl"
)
```

RC4 的加密和解密都使用同一段密钥流。按反编译代码还原 KSA 与 PRGA，先做 Base64 解码，再异或即可：

```python
import base64


def keystream(key: str, length: int) -> list[int]:
    state = list(range(256))
    j = 0

    for i in range(256):
        j = (j + state[i] + ord(key[i % len(key)])) % 256
        state[i], state[j] = state[j], state[i]

    i = j = 0
    result = []
    for _ in range(length):
        i = (i + 1) % 256
        j = (j + state[i]) % 256
        state[i], state[j] = state[j], state[i]
        result.append(state[(state[i] + state[j]) % 256])

    return result


key = "As_we_do_as_you_know"
enc = (
    "wr3ClVcSw7nCmMOcHcKgacOtMkvDjxZ6asKWw4nChMK8IsK7KMOOasOrdgbDlx3Dqc"
    "Kqwr0hw701Ly57w63CtcOl"
)

# 原程序先把异或结果作为 UTF-8 字符串编码，再做 Base64。
ciphertext = base64.b64decode(enc).decode()
plain = "".join(
    chr(ord(char) ^ key_byte)
    for char, key_byte in zip(ciphertext, keystream(key, len(ciphertext)))
)
print(plain)
```

输出为：

```text
hgame{python_reverse_is_easy_with_internet}
```

上述脚本已对 PDF 截图中的密钥、密文与算法重新运行验证。文件列表、在线反编译器和 CyberChef 界面截图仅展示工具操作，已转写为命令与代码，不作为图片保留。

## 方法总结

本题的关键路径是“识别 PyInstaller → 提取入口 `.pyc` → 恢复 RC4 生成器 → 逆向 Base64 与异或顺序”。分析打包 Python 时，应先确认嵌入的 Python 版本，再选择兼容的反编译器；拿到源码后仍要按程序真实的字符串编码顺序复现，不能只凭“看起来像 RC4”直接套标准工具参数。
