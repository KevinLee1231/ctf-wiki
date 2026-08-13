# Overly Simplified Rev Challenge

## 题目简述

题目通过内联所有逻辑、移除普通栈用法并避免常见 `mov` 指令来破坏反编译体验，但 flag 校验仍是可逆的数据流水线：修改初始 S 盒的 RC4、前缀异或链和逐字节异或 `0x80`。从固定地址读出 32 字节密钥与 40 字节密文，按相反顺序撤销即可。

## 解题过程

校验程序先要求输入长度为 40 且前五字节为 `grey{`。随后使用一个变体 RC4：标准算法从 `[0,1,...,255]` 初始化 S 盒，而这里从 `[255,254,...,0]` 开始。RC4 输出之后再做：

```c
for (i = 1; i < 40; i++)
    ct[i] ^= ct[i - 1];
```

最后，字符串比较函数又把目标数组每字节异或 `0x80`。逆向时先从 `0x404020` 读出 40 字节目标，撤销 `0x80`，再从后往前撤销前缀链，最后用同一个 RC4 变体处理一次：

```python
from pwn import ELF, xor

elf = ELF("./chall")
key = elf.read(0x4010a0, 0x20)
data = bytearray(xor(elf.read(0x404020, 40), 0x80))

for i in range(len(data) - 1, 0, -1):
    data[i] ^= data[i - 1]

def rc4_variant(data, key):
    s = list(range(255, -1, -1))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xff
        s[i], s[j] = s[j], s[i]
    i = j = 0
    out = bytearray()
    for c in data:
        i = (i + 1) & 0xff
        j = (j + s[i]) & 0xff
        s[i], s[j] = s[j], s[i]
        out.append(c ^ s[(s[i] + s[j]) & 0xff])
    return bytes(out)

print(rc4_variant(data, key).decode())
```

得到：

```text
grey{wasnt_that_fun_and_easy_XDXDXDXDXD}
```

## 方法总结

反编译器失效时，优先追踪输入缓冲区、固定数据和最终比较点，而不是试图恢复漂亮的伪代码。这里三层变换都可逆；尤其要注意前缀异或链必须从末尾向前撤销，否则会覆盖下一步所需的旧值。
