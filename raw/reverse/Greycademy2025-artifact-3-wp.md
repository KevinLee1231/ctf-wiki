# Artifact 3

## 题目简述

附件是 stripped、PIE 的 x86-64 ELF。程序要求输入恰好 20 字节，先原地变换输入，再与固定 20 字节目标比较。前 15 字节统一减 3，后 5 字节分别减去字符串 `HELLO`，所以要逆向这些可逆变换恢复原输入。

## 解题过程

核心检查可整理为：

```c
if (strlen(input) != 20)
    return wrong;
for (int i = 0; i < 15; i++)
    input[i] -= 3;
for (int i = 0; i < 5; i++)
    input[15 + i] -= "HELLO"[i];
return strcmp(input, target) == 0;
```

目标字节不能完全按可打印字符串抄写，因为其中含 `0x1B`：

```text
64 6f 62 76 78 6f 62 5e 69 5c 6f 62 73 62 6f 2b 24 22 1b 2e
```

逐字节执行逆运算：

```python
target = bytes.fromhex(
    "64 6f 62 76 78 6f 62 5e 69 5c 6f 62 73 62 6f "
    "2b 24 22 1b 2e"
)
key = b"HELLO"

plain = bytes(x + 3 for x in target[:15])
plain += bytes(x + k for x, k in zip(target[15:], key))
print(plain.decode())
```

脚本输出并经原程序验证：

```text
grey{real_reversing}
```

## 方法总结

看到“先修改输入、再 `strcmp`”时，比较常量并不是应直接提交的输入，必须逆转修改函数。不可打印目标字节要从十六进制视图或原始数据读取，避免反编译字符串转义造成丢字节；无效除法和空函数只是混淆噪声，不应进入最终模型。
