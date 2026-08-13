# 67

## 题目简述

附件是一个以大小写变化的 `six`、`seven` 词汇伪装指令名的 Python 虚拟机。程序以模 67 的幂排列访问输入字符，再与线性同余状态异或比较。目标是还原指令语义和检查顺序，逆推出 67 字节 flag。

## 解题过程

把混淆后的类方法与未混淆版本逐项对应，可以恢复 `LOAD`、`STORE`、`MOV`、`ADD`、`SUB`、`MUL`、`XOR`、`MOD`、`READC`、`PUTS`、`JNZ` 和 `HALT`。真正的校验可概括为：

```python
state = 76
for i in range(67):
    index = pow(13, i + 1, 67)
    state = state * 67 + 86
    assert ((flag[index] ^ state) & 0xff) == expected[i]
```

由于最终只保留低 8 位，只需用 `state & 0xff` 逆异或。`13` 在模 67 的非零剩余类中生成一个长度为 66 的循环，因此访问集合恰好是下标 1 到 66；第 67 轮只是重复其中一个下标，下标 0 根本没有被检查。结合 flag 格式把首字节设为 `g`：

```python
expected = bytes.fromhex(
    "54f7ac0f835197c523b75dafb2ef77652c17474455ac5709c5ca3ad5520b0d"
    "a5c75b07a315e2210d8c45dd0f0f71fcdfacadfdc4cf07f7bf69779d59fc91"
    "9c026cf68c"
)

state = 76
flag = bytearray(b"?" * 67)
flag[0] = ord("g")

for i, value in enumerate(expected):
    index = pow(13, i + 1, 67)
    state = state * 67 + 86
    flag[index] = value ^ (state & 0xff)

print(flag.decode())
```

输出为：

```text
grey{Six_SeVen_SIx_sEVeN_six_SEVen_siX_SEVeN_sIx_SeVeN_SiX_sEVen!!}
```

将该字符串输入分发版虚拟机得到 `Correct!`。把首字节换成其他字符也能通过，进一步验证了下标 0 未参与校验。

## 方法总结

逆向自定义 VM 时，先恢复小而稳定的指令语义，再把长指令流压缩成数学关系。本题最容易忽略的是访问序列的覆盖范围：模幂循环只遍历非零剩余类，导致首字节未校验。对任何置换式索引都应显式统计周期和覆盖集合，而不能只看循环次数。
