# 虚构核心

## 题目简述

题目附件是一个 Android APK。主程序没有直接实现 flag 校验，而是解密资源中的 `encrypted.dex`，动态加载 `com.ctf.a0xgame_5.FlagChecker`，再调用其 `checkFlag(String)` 方法。

## 解题过程

### 解密并加载 DEX

反编译 `MainActivity` 后，可以确定完整调用链：读取 `assets/encrypted.dex`，使用密钥 `The0xGameKey` 循环异或，然后动态加载解密后的 DEX，并反射调用 `FlagChecker.checkFlag`。

无需运行应用，可直接从 APK 提取并解密该资源：

```python
from pathlib import Path
import zipfile

key = b"The0xGameKey"

with zipfile.ZipFile("虚构核心.apk") as apk:
    encrypted = apk.read("assets/encrypted.dex")

decrypted = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(encrypted)
)
Path("decrypted.dex").write_bytes(decrypted)
```

将 `decrypted.dex` 交给 JADX，便可看到真正的校验逻辑。

### 恢复三段十六进制字符串

`checkFlag` 要求输入长度为 44，以 `0xGame{` 开头、以 `}` 结尾；花括号中的内容按 `_` 分成五段：

- 第一段固定为 `f5bf50a3`；
- 第五段固定为 `f3eddaccb39f`；
- 中间三段均由 4 个十六进制字符组成，并分别匹配三个 MD5 值。

由于每段只有 $16^4=65536$ 种可能，可以在本地穷举，无需依赖外部哈希查询站点：

```python
import hashlib
import itertools

targets = {
    "69b4fa3be19bdf400df34e41b93636a4",
    "76b662c5c3d6d98035190115d89ef42f",
    "87fff610a9c97ebbe5a16a6d4865c0e4",
}
found = {}

for chars in itertools.product("0123456789abcdef", repeat=4):
    candidate = "".join(chars)
    digest = hashlib.md5(candidate.encode()).hexdigest()
    if digest in targets:
        found[digest] = candidate
        if len(found) == len(targets):
            break

for target in targets:
    print(target, found[target])
```

三个原文依次为 `9988`、`61ee`、`8c00`，拼接得到：

```text
0xGame{f5bf50a3_9988_61ee_8c00_f3eddaccb39f}
```

## 方法总结

该题的关键是沿动态加载链找到真正的校验代码：外层 APK 只负责用固定密钥循环异或解密 DEX，内层 `FlagChecker` 才包含 flag 结构和 MD5 条件。四位十六进制字符串的搜索空间很小，直接本地穷举既可复现，也避免依赖外部反查结果。
