# 删库跑路

## 题目简述

附件解压后只剩 `.git/` 目录，工作区文件看似已经被删除。`.git` 保存了提交、树和文件对象，只要相关对象没有被垃圾回收，仍可从版本库历史中恢复源码与输出。本题恢复出的 `main.py` 对 flag 依次执行凯撒移位、单字节异或和 Base64 编码，最终密文保存在 `output` 中。

需要注意，Git 松散对象通常经过 zlib **压缩**，不是所谓的“zlib 加密”；恢复的关键是让 Git 根据对象关系还原提交内容。

## 解题过程

进入包含 `.git` 的目录，用 Git 原生命令查看所有分支和提交：

```bash
git log --all --oneline
```

可以看到两个提交：

```text
c5bf454 initial
79f2853 save output
```

无需覆盖当前目录，直接从 `HEAD` 读取被记录的文件：

```bash
git show HEAD:main.py
git show HEAD:output
```

`main.py` 的加密顺序为：

1. 对英文字母执行移位量为 `114514` 的凯撒加密；
2. 对每个字节异或 `0x66`；
3. 对结果进行 Base64 编码。

恢复出的 `output` 内容为：

```text
Vg43DREJHUgXFQI5EAkNEzkVBTkgCQQPAAkEDzklKQRXHwMFR0dHGw==
```

解密时按相反顺序执行。凯撒移位只与 `114514 mod 26 = 10` 有关：

```python
import base64

XOR_KEY = 0x66
CAESAR_SHIFT = 114514


def xor_bytes(data: bytes, key: int) -> bytes:
    return bytes(value ^ key for value in data)


def caesar_decrypt(text: str, shift: int) -> str:
    result = []
    for char in text:
        if "A" <= char <= "Z":
            result.append(
                chr((ord(char) - ord("A") - shift) % 26 + ord("A"))
            )
        elif "a" <= char <= "z":
            result.append(
                chr((ord(char) - ord("a") - shift) % 26 + ord("a"))
            )
        else:
            result.append(char)
    return "".join(result)


with open("output", "r", encoding="ascii") as file:
    ciphertext = file.read().strip()

decoded = base64.b64decode(ciphertext)
xored = xor_bytes(decoded, XOR_KEY).decode("utf-8")
flag = caesar_decrypt(xored, CAESAR_SHIFT)
print(flag)
```

运行后得到：

```text
0xGame{.git_leak_is_Veryvery_SEr1ous!!!}
```

## 方法总结

遇到仅删除工作区、却遗留 `.git` 的场景，应先用 `git log --all` 确认提交，再用 `git show <提交>:<路径>` 只读恢复目标文件。拿到加密源码后，严格逆序处理 Base64、异或和凯撒移位即可；不要把 Git 对象的 zlib 压缩误判为密码算法。
