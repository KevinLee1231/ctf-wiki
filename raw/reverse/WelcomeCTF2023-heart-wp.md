# ❤

## 题目简述

附件是一段高度排版混淆的 Python 程序。代码绘制 Mandelbrot 风格的 ASCII 动画，同时把 `__file__` 的文件名参与 SHA-1 和异或运算；文件名不正确时只显示干扰内容，正确文件名才会走到 Flag 解密分支。

## 解题过程

展开最外层 lambda 后，可以观察到以下关键数据流：

```python
f = [sys.argv[0].encode()]
h = hashlib.sha1()
f[0] = xor(f[0], h.digest())
```

也就是说，程序不是只校验输入内容，而是在迭代过程中持续使用脚本自身路径。官方 README 给出的正确文件名为：

```text
EAT-THIS-SUCKERRR.py
```

把附件改为该名称后直接运行：

```bash
cp '❤.py' EAT-THIS-SUCKERRR.py
python3 EAT-THIS-SUCKERRR.py
```

正确文件名使最终摘要值匹配硬编码常量，程序随后异或并打印：

```text
greyhats{N0oo00o0! My 0nly We4ken3ss! Dy1ng!}
```

## 方法总结

- 核心技巧：去除 lambda/排版混淆后跟踪 `__file__`，识别“文件名即密钥材料”的校验。
- 识别信号：混淆脚本读取 `sys.argv[0]` 或 `__file__`，且执行结果随重命名变化。
- 复用要点：分析自引用程序时要区分文件内容、文件名和完整路径；测试时最好在固定目录使用题目要求的精确大小写名称。
