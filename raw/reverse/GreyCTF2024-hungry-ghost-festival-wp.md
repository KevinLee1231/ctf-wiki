# Hungry Ghost Festival

## 题目简述

64 位主程序只有在指定 Unix 时间戳且未被 `ptrace` 调试时才映射 `gate.bin`，随后通过 far return 切换到 32 位兼容模式执行 shellcode，也就是 Heaven's Gate。shellcode 根据时间与主程序 MD5 派生 32 位异或密钥并解密 32 字节正文；静态恢复最终密钥后可直接解开文件末尾的密文。

## 解题过程

主程序的门控条件是：

```c
if (time(NULL) == 1722700800 && ptrace(PTRACE_TRACEME) != -1) {
    open_the_gate_of_hell();
}
```

`open_the_gate_of_hell` 把 `gate.bin` 以 `MAP_32BIT` 映射为可读、可写、可执行内存，将自身 MD5 的前 4 个 ASCII 字节写入偏移 `0x500`，再用 `memfrob` 对整页异或 42。最后构造代码段选择子 `0x23` 并执行 `retf`，从 x86-64 切入 32 位 shellcode。

解码后的 shellcode 调用 `gettimeofday`，把秒数与先前注入的数据异或，得到最终 32 位循环密钥 `0x7be61b4b`。生成端把不含 `grey{}` 的 32 字节正文先与该密钥循环异或，再额外与 42 异或后放在 `gate.bin` 的末尾。因此可以绕过时间、反调试和模式切换，直接复现最终数据流：

```python
from pathlib import Path

blob = Path("gate.bin").read_bytes()
encrypted = bytes(x ^ 42 for x in blob[-32:])
key = (0x7be61b4b).to_bytes(4, "little")
body = bytes(x ^ key[i % 4] for i, x in enumerate(encrypted))
print(b"grey{" + body + b"}")
```

输出为：

```text
grey{tr1ppy_4ss_r3v3rs1ng_ch4ll3ngexd}
```

## 方法总结

Heaven's Gate、时间门控与 `ptrace` 都服务于隐藏实际 shellcode，但 flag 最终仍是固定密文和固定派生密钥的异或。分析时应把执行环境障碍与核心数据变换分开：确认 `memfrob`、低地址映射和 far return 的作用后，沿密文生成链静态复现即可。
