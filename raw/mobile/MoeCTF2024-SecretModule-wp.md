# SecretModule

## 题目简述

附件是一个 Magisk 模块。安装脚本 `customize.sh` 先用 Base64 和 bzip2 包住真正逻辑，再通过 Android 的 `getevent` 读取音量键事件。题目的关键不在传统 native 逆向，而在 Magisk 安装脚本、Android 输入事件与一个只有 $2^7$ 种输入的 MD5 校验，因此归入 Mobile。

## 解题过程

去掉 `customize.sh` 外层的 `eval "$(printf ... | base64 -d | bunzip2 -c)"` 后，可以看到核心有两个函数：

- `testk` 要求设备能通过 `/system/bin/getevent` 捕获一次带有 `VOLUME` 和 `DOWN` 的按键事件；
- `choose` 把 `VOLUMEUP` 映射为字符串 `114514`，把另一个音量键映射为 `1919810`。

安装脚本连续读取七次按键，把七段字符串直接拼接，然后比较

```bash
md5sum(concatenated) == 77a58d62b2c0870132bfe8e8ea3ad7f1
```

每一位只有两种选择，直接枚举 $2^7=128$ 种组合即可，无须真的反复刷入模块：

```python
from hashlib import md5
from itertools import product

target = "77a58d62b2c0870132bfe8e8ea3ad7f1"
choices = ("114514", "1919810")

for keys in product(choices, repeat=7):
    candidate = "".join(keys)
    if md5(candidate.encode()).hexdigest() == target:
        print(keys)
        print(f"moectf{{{candidate}}}")
        break
```

命中的按键序列是：

```text
VOLUMEUP, VOLUMEUP, VOLUMEDOWN, VOLUMEUP,
VOLUMEUP, VOLUMEDOWN, VOLUMEDOWN
```

对应结果为：

```text
moectf{114514114514191981011451411451419198101919810}
```

## 方法总结

这题的有效分析顺序是先识别 Magisk 模块入口，再解开 Shell 包装，最后把物理按键交互抽象成二元枚举。`getevent -lc 1` 每次只读取一个输入事件；七次选择空间极小，MD5 在这里仅是验证函数，并不需要进行通用哈希破解。
