# before_main

## 题目简述

程序实现了一个使用自定义字母表的 Base64 校验，但真正的字母表会在 `main` 执行前被替换。初始化函数还调用 `ptrace(PTRACE_TRACEME, ...)` 检测调试器，导致直接动态调试时可能看不到换表操作。

## 解题过程

交叉引用初始化代码可以发现，它被 GCC 的 `__attribute__((constructor))` 标记，因此会在 `main` 前由运行时调用。其关键行为可概括为：

```c
if (ptrace(PTRACE_TRACEME, 0, 0, 0) != -1) {
    strcpy(base64_table,
        "qaCpwYM2tO/RP0XeSZv8kLd6nfA7UHJ1No4gF5zr3VsBQbl9juhEGymc+WTxIiDK");
}
```

处于已被跟踪的调试状态时，`ptrace` 返回 `-1`，换表分支不会执行。因此需要静态读取构造函数，或在反调试检查前后下断点并修正返回值。

主逻辑中的待解码字符串为：

```text
AMHo7dLxUEabf6Z3PdWr6cOy75i4fdfeUzL17kaV7rG=
```

将自定义字母表中的字符映射回标准 Base64 字母表，再正常解码即可：

```python
import base64

standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
custom = "qaCpwYM2tO/RP0XeSZv8kLd6nfA7UHJ1No4gF5zr3VsBQbl9juhEGymc+WTxIiDK"
ciphertext = "AMHo7dLxUEabf6Z3PdWr6cOy75i4fdfeUzL17kaV7rG="

translated = ciphertext.translate(str.maketrans(custom, standard))
flag = base64.b64decode(translated).decode()
print(flag)
```

输出为：

```text
hgame{s0meth1ng_run_befOre_m@in}
```

该结果已用上述映射脚本重新计算验证；PDF 中的 IDA 与 CyberChef 截图均可由代码和输出完整替代，因此没有保留为图片。

## 方法总结

本题的两个干扰点分别是 `constructor` 早期初始化和 `ptrace` 反调试。逆向时不能只从 `main` 开始阅读，还应检查 `.init_array`、构造函数和全局对象初始化；发现“静态字母表无法解码”时，应继续追踪该数据的写引用，而不是假定算法识别错误。
