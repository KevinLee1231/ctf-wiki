# L3akCTF 2025 BrainCalc Writeup

## 题目简述

BrainCalc 表面上是一款 Android 心算练习应用，题目附件为 `app-debug.apk`。应用本身由 BeeWare/Chaquopy 打包，真正的答题与奖励逻辑保存在 APK 资源中的 Python 字节码里。

虽然题目在比赛中归为 Mobile，但获取 flag 的决定性步骤是识别 Python 打包结构、提取 `.pyc` 并恢复其逻辑，因此本文按 Reverse 归档。

## 解题过程

### 定位 Python 资源

APK 本质上是 ZIP 容器。列出文件后，可以在 `assets/chaquopy/` 下找到 `app.imy`：

```powershell
Expand-Archive -LiteralPath "app-debug.apk" -DestinationPath "braincalc-apk"
Get-ChildItem -LiteralPath "braincalc-apk/assets/chaquopy" -Recurse
```

`app.imy` 同样是 ZIP 格式。将其解包后得到：

```text
BrainCalc/app.pyc
```

该文件的 magic number 为 3531，对应 CPython 3.12 字节码。这个版本信息很重要：使用不支持 3.12 opcode 的反编译器时，容易得到缺失函数体或异常控制流。

### 反编译奖励函数

使用支持相应字节码版本的 `pycdc` 处理文件：

```bash
/home/kali/pycdc/build/pycdc BrainCalc/app.pyc
```

反编译结果中的 `check_answer` 含生成器相关指令，工具未能完整恢复，但 flag 不在该函数的复杂控制流中。`get_secret_reward()` 已被完整还原：

```python
def get_secret_reward():
    compressed_flag = "eJzzMXb0rvYqLS6JN4kPNynKjQ8tiHfOMMnJqQUAeHcJQA=="
    decoded = base64.b64decode(compressed_flag)
    flag = zlib.decompress(decoded).decode("utf-8")
    return flag
```

这段代码先对字符串做 Base64 解码，再使用 zlib 解压：

```python
import base64
import zlib

blob = "eJzzMXb0rvYqLS6JN4kPNynKjQ8tiHfOMMnJqQUAeHcJQA=="
flag = zlib.decompress(base64.b64decode(blob)).decode("utf-8")
print(flag)
```

运行后得到：

```text
L3AK{Just_4_W4rm_Up_Ch4ll}
```

### 结果核验

仓库中的官方解题说明指出，应从 `app.imy` 中提取并反编译 `app.pyc`。实际 APK 内的路径、CPython 版本以及上述解码结果均与该说明和题目给出的 flag 一致。

## 方法总结

这道题的关键不是尝试通过心算界面逐题通关，而是识别 APK 中的 Chaquopy Python 运行环境。看到 `assets/chaquopy`、`.imy` 和 `.pyc` 时，应优先把各层容器解开，再选择支持对应 CPython 版本的反编译工具。

即使反编译器无法完整恢复某些新版本字节码，也不必立即转向动态调试。应先检查独立函数、常量和标准库调用；本题的奖励函数只包含 Base64 与 zlib 两层可逆处理，已经足以直接、可复现地恢复 flag。
