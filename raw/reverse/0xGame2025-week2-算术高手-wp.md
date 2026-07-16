# 算术高手

## 题目简述

附件 `babyPy.exe` 是 64 位 Windows 控制台程序。DIE 能识别出 PyInstaller 打包特征，解包结果中还包含 `python38.dll`，说明入口逻辑来自 Python 3.8 字节码，而不是需要逐函数逆向的原生 C/C++ 程序。

程序表面上是 10 道乘法题，每题答对加 10 分，声称达到 100 分即可获得 flag；实际计分逻辑故意让正常游戏路径无法满足最终条件。

## 解题过程

用 [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) 提取 PyInstaller 内嵌归档。该工具会拆出 CArchive/PYZ 内容，并修复 `pyc` 头，使字节码反编译器能够识别：

```bash
python3 pyinstxtractor.py babyPy.exe
```

入口文件位于：

```text
babyPy.exe_extracted/babyPy.pyc
```

再用与 Python 3.8 字节码兼容的反编译器恢复源码；在当前分析环境中可直接调用：

```bash
/home/kali/pycdc/build/pycdc babyPy.exe_extracted/babyPy.pyc
```

原 PDF 另给出在线 [PyLingual](https://www.pylingual.io/) 作为反编译选择。在线服务不是复现的必要条件；无论使用哪个工具，都必须先确认 `python38.dll` 指示的字节码版本，并以恢复出的控制流和常量交叉验证结果。

flag 作为普通字符串常量直接保存在模块顶层。计分部分的关键逻辑为：

```python
flag = "0xGame{c2a6d59d-34dc-4b94-96aa-e823bdcb4823}"
score = 0

for i in range(10):
    # 每题读取两个 1000 至 10000 的随机数并检查乘法答案。
    if answer_is_correct:
        score += 10

    if score >= 100:
        input("忘跟你说坏消息了，你的计分器只有两位数")
        score %= 100

if score == 100:
    print(f"恭喜你，满分！flag是：{flag}")
else:
    print("很遗憾，得分不够！")
```

十题全对时，分数轨迹为 10、20、……、90、100；但第十题后立刻执行 `100 % 100 = 0`。故意答错又只能得到小于 100 的分数，所以最终 `score == 100` 在正常交互中不可达。无需完成算术题，直接读取模块常量即可得到：

```text
0xGame{c2a6d59d-34dc-4b94-96aa-e823bdcb4823}
```

## 方法总结

PyInstaller 只是把解释器、依赖和 Python 字节码封装进 PE，并不会自动隐藏源码常量。识别封装格式后，应解包并针对 `pyc` 版本选择兼容反编译器。本题真正的陷阱是先对满分取模、再判断是否满分，造成不可达分支；正文已经保留入口路径、关键逻辑和 flag，因此不需要依赖在线反编译网站或工具截图。
