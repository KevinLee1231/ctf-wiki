# UIUCTF 2023 VMWhere 2 Writeup

## 题目简述

VMWhere 2 沿用第一题的栈式虚拟机，并新增两个位操作指令：

```text
SCAT：弹出一个字节，按位拆成 8 个栈元素
COAL：弹出 8 个位元素，重新合并为一个字节
```

字节码不再直接做半字节异或，而是把每个输入字符的二进制位送入一段循环，构造类似三进制表示的数值；之后从后向前与常量比较，并把所有比较结果 OR 到一起。

## 解题过程

先在第一题指令表上补充 `0x11 = SCAT`、`0x12 = COAL`，即可完整反汇编 `program`。每个输入字符的第一阶段结构为：

```text
IN
SCAT
PUSH 255
REV 9
REV 8
PUSH 0
...逐位循环...
```

哨兵 `0xff` 标记位序列结束。循环的高层等价逻辑是：

```python
def encode_char(x: int) -> int:
    value = 0
    for bit in bin(x)[2:]:
        if bit == "1":
            value += 1
        value *= 3
    return value & 0xff
```

也就是对每一位执行 $v\leftarrow3(v+b)$。比较阶段按字符逆序出现，典型结构为：

```text
PUSH <expected>
XOR
REV n
REV n+1
OR
...
```

`PUSH` 只有 1 字节立即数，因此生成器算出的三进制式常量最终只保留低 8 位。该映射不是全域一一对应，但 flag 使用的字符集很窄；枚举小写字母、数字、下划线和花括号即可为每个位置找到唯一字符。下面的脚本直接从原始字节码抽取 `PUSH expected; XOR; REV` 模板：

```python
from hashlib import sha256
from pathlib import Path

code = Path("program").read_bytes()

expected = []
for i in range(len(code) - 3):
    # 0x0a=PUSH, 0x05=XOR, 0x10=REV
    if code[i] == 0x0a and code[i + 2] == 0x05 and code[i + 3] == 0x10:
        expected.append(code[i + 1])

# 校验块按 flag 的逆序生成
expected.reverse()

def encode_char(x: int) -> int:
    value = 0
    for bit in bin(x)[2:]:
        if bit == "1":
            value += 1
        value *= 3
    return value & 0xff

alphabet = "abcdefghijklmnopqrstuvwxyz0123456789_{}"
flag = ""
for target in expected:
    candidates = [c for c in alphabet if encode_char(ord(c)) == target]
    assert len(candidates) == 1, (target, candidates)
    flag += candidates[0]

assert sha256(flag.encode()).hexdigest() == \
    "4c17f2be4c4d2759c4195f075019331fa3bd68a03f1dd04c10023cbc134d82ae"
print(flag)
```

程序提取 46 个检查常量，得到：

```text
uiuctf{b4s3_3_1s_b4s3d_just_l1k3_vm_r3v3rs1ng}
```

再交给原解释器验证：

```text
$ printf '%s' 'uiuctf{b4s3_3_1s_b4s3d_just_l1k3_vm_r3v3rs1ng}' | ./chal program
Welcome to VMWhere 2!
Please enter the password:
Correct!
```

## 方法总结

第二题的难点是识别新增位操作后跨多条指令的数据流。将重复循环提升为单个 `encode_char()`，再注意一字节立即数带来的模 $256$ 截断，复杂 VM 汇编就会退化为逐字符逆像搜索。映射存在碰撞时，应显式限定题目允许的字符集，并用题面给出的 SHA-256 校验最终候选，而不是凭 flag 外观猜测。
