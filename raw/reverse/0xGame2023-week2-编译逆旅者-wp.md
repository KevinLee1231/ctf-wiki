# 编译逆旅者

## 题目简述

附件是 Python 3.11 的 `.pyc` 字节码。反编译或反汇编后可以看到：程序要求一个 13 位纯数字字符串，与常量 `1145141919810` 比较；匹配后把内置大整数转换为十六进制字节并输出 flag。

## 解题过程

反编译结果能够识别 Python 版本、输入长度检查和比较常量。在线反编译器对 Python 3.11 新字节码的变量恢复存在误差，曾把比较左侧错误恢复为 `None`；不过常量 `1145141919810` 和成功分支仍可辨认。更稳妥的本地方法是在相同 Python 3.11 环境中跳过 16 字节 pyc 头，加载 code object 后用标准库反汇编：

```python
import dis
import marshal

with open("challenge.pyc", "rb") as f:
    f.read(16)  # Python 3.7+ 常见 pyc 头长度
    code = marshal.load(f)

dis.dis(code)
for const in code.co_consts:
    if hasattr(const, "co_code"):
        dis.dis(const)
```

从字节码常量得到秘密数字 `1145141919810`。成功分支把一个大整数按十六进制还原为字节；等价的本地验证代码如下：

```python
encoded_flag = 0x307847616D657B63646539646331372D356133312D356330612D646633342D36633735623736346334627D
flag = bytes.fromhex(f"{encoded_flag:x}").decode()
print(flag)
```

用 Python 3.11 直接运行 pyc 并输入秘密数字，或执行上面的等价代码，均得到：

```text
0xGame{cde9dc17-5a31-5c0a-df34-6c75b7646c4b}
```

## 方法总结

`.pyc` 包含 code object、常量表和字节码，并不保护业务秘密。反编译失败时应回退到匹配版本的 `marshal` 与 `dis`，重点检查字符串/整数常量和比较跳转；在线上传工具并非必要，还可能泄露未公开附件。
