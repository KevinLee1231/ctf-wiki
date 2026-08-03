# UIUCTF 2023 VMWhere 1 Writeup

## 题目简述

附件包含一个解释器 `chal` 和独立字节码文件 `program`。解释器实现基于字节栈的虚拟机，主要指令包括 `PUSH`、`IN`、`OUT`、`XOR`、移位、相对跳转、复制与栈区反转。程序逐字节读取密码，全部校验通过后输出 `Correct!`。

主障碍是识别 VM 指令格式并把密码校验还原为可逆的逐字节关系。

## 解题过程

### 恢复指令集

反编译 `chal`，在基址 `0x1000` 附近可找到解释循环；核心 `switch` 位于约 `0x144c`。各条指令长度为：

- 普通指令 1 字节；
- `PUSH`、`REV` 后跟 1 字节立即数；
- `JS`、`JZ`、`JMP` 后跟 2 字节大端有符号相对偏移，目标以当前指令末尾为基准。

据此顺序扫描 `program` 即可得到汇编。提示字符串之后，每个输入字符都出现同一段结构：

```text
IN
DUP
PUSH 4
SHR
XOR
XOR
DUP
PUSH <累计常量>
XOR
JZ <next>
JMP <fail>
POP
```

设输入字节为 $x_i$，先计算：

$$
t_i=x_i\oplus(x_i\gg4).
$$

栈底保存前面所有 $t$ 的累计异或，所以第 $i$ 次比较的常量为：

$$
k_i=t_0\oplus t_1\oplus\cdots\oplus t_i.
$$

因此先用 $t_i=k_i\oplus k_{i-1}$ 消去累计关系。对一个字节，映射 $t=x\oplus(x\gg4)$ 也可直接求逆：高半字节不变，低半字节再与高半字节异或一次，故

$$
x=t\oplus(t\gg4).
$$

### 自动提取

下面的脚本直接在字节码中定位固定检查模板，提取其中的累计常量并逐字节求逆：

```python
from pathlib import Path

code = Path("program").read_bytes()
signature = bytes([0x08, 0x0f, 0x0a, 0x04, 0x07,
                   0x05, 0x05, 0x0f, 0x0a])

keys = []
for i in range(len(code) - 10):
    # 模板末尾应为 PUSH key; XOR
    if code[i:i + len(signature)] == signature and code[i + 10] == 0x05:
        keys.append(code[i + 9])

previous = 0
flag = bytearray()
for current in keys:
    transformed = current ^ previous
    flag.append(transformed ^ (transformed >> 4))
    previous = current

print(flag.decode())
```

脚本提取 57 个常量，输出：

```text
uiuctf{ar3_y0u_4_r3al_vm_wh3r3_(gpt_g3n3r4t3d_th1s_f14g)}
```

把结果送入原程序可复核：

```text
$ printf '%s' 'uiuctf{ar3_y0u_4_r3al_vm_wh3r3_(gpt_g3n3r4t3d_th1s_f14g)}' | ./chal program
Welcome to VMWhere 1!
Please enter the password:
Correct!
```

## 方法总结

自定义 VM 题应先确定栈元素宽度、指令长度、立即数端序和跳转基准，再编写最小反汇编器。重复出现的基本块通常直接暴露校验的代数结构；本题的累计异或并没有增加不可逆性，只需相邻常量再异或一次即可拆成独立字符，随后逆掉半字节折叠。
