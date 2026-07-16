# FirstSight-Pyc

## 题目简述

附件是 Python 3.8 生成的 `.pyc`。反编译后可见程序只接受固定输入 `Ciallo~`，计算其 MD5 十六进制摘要，再对摘要中的十进制数字执行 ROT5，最后加上 `0xGame{}` 外壳。

## 解题过程

使用支持 Python 3.8 字节码的 pycdc、uncompyle6 等工具反编译，可恢复核心逻辑：

```python
import hashlib

user_input = input("请输入神秘代号：")
if user_input != "Ciallo~":
    print("代号不是这个哦")
    raise SystemExit

digest = list(hashlib.md5(user_input.encode()).hexdigest())
for index, char in enumerate(digest):
    if "0" <= char <= "9":
        digest[index] = str((int(char) + 5) % 10)

print("0xGame{{{}}}".format("".join(digest)))
```

无需依赖 Python 3.8 直接运行原 pyc，也可以按同一算法重算：

```python
import hashlib

digest = hashlib.md5(b"Ciallo~").hexdigest()
encoded = "".join(
    str((int(char) + 5) % 10) if char.isdigit() else char
    for char in digest
)

print(digest)
print(f"0xGame{{{encoded}}}")
```

输出：

```text
7f5ef5762bf8a2c043d836b522127e54
0xGame{2f0ef0217bf3a7c598d381b077672e09}
```

## 方法总结

处理 pyc 时先确认字节码版本，再选择兼容的反编译器；版本不匹配通常比算法本身更容易造成失败。本题恢复源码后只需复现“MD5 + 数字 ROT5”，不必保留反编译和终端的纯文本截图。
