# Is this really python?

## 题目简述

附件虽名为 `.py`，实际是由 Python 3.8 程序经 PyInstaller 打包的 ELF。内部校验逻辑使用复杂列表推导式：先把输入字符与索引异或、把十进制结果串联，再做游程编码。目标是提取字节码、简化算法并逆变换。

## 解题过程

`strings` 会出现 PyInstaller 的归档错误信息，可据此识别打包器。先用 `pyinstxtractor` 解出归档，再用与原程序匹配的 Python 3.8 环境反编译 `challenge.pyc`：

```bash
python3 pyinstxtractor.py challenge.py
uncompyle6 challenge.py_extracted/challenge.pyc
```

整理后的校验可概括为：

1. 对第 $i$ 个字符计算 `ord(ch) ^ i`；
2. 把每个结果的十进制表示无分隔拼接；
3. 对连续相同数字做“次数 + 数字”的游程编码；
4. 与硬编码目标串比较。

官方求解先逆转游程编码并人工确认十进制数的边界，得到异或后的整数数组，再按索引异或回原字符：

```python
encoded_values = [103, 115, 103, 122, 127, 108, 89, 112]  # 后续省略
flag = "".join(chr(i ^ value) for i, value in enumerate(encoded_values))
```

完整恢复结果为：

```text
grey{i_wrote_this_at_1am_and_i_have_no_idea_what_i_just_cooked}
```

## 方法总结

PyInstaller 并未把 Python 逻辑变成不可恢复的原生算法，它主要封装解释器和字节码。处理这类附件应先识别打包器、匹配字节码版本、反编译，再把嵌套推导式重构成清晰的逐步变换后求逆。
