# Reversing 101

## 题目简述

题目给出一个未剥离的 x86-64 ELF 密码检查程序，并要求回答六个逆向问题后取得 flag。检查逻辑被拆成三个函数：`a` 计算输入长度，`b` 用大量无关运算生成一个固定的 64 位返回值，`c` 用该返回值作为密钥加密输入，再与 15 字节常量比较。真正需要恢复的是主函数地址、三个函数的语义以及正确密码。

## 解题过程

程序没有 PIE，因此反汇编中 `main` 的地址固定为 `0x402db6`。函数 `a` 从首字符开始递增指针，直至遇到 `\0`，返回首尾指针差，语义等同于 `strlen`；主函数要求其返回值为 `0xf`，所以密码长度是 15。

函数 `b` 虽然包含大量状态混合，但没有外部输入。可以在调用返回后读取 `RAX`，得到常量：

```text
0xc1de1494171d9e2f
```

函数 `c` 先初始化 `S[0..255]`，执行 RC4 KSA，再逐字节执行 PRGA 并与输入异或。密钥是上面这个 64 位整数的小端 8 字节表示。将程序中的密文数组反向执行同一 RC4 流即可恢复密码：

```python
enc = bytes([
    0xD1, 0x58, 0x15, 0x8A, 0xEE, 0xB5, 0xBB, 0x52,
    0x0C, 0x6B, 0xA4, 0xAB, 0x6D, 0x7D, 0xB7,
])
key = (0xC1DE1494171D9E2F).to_bytes(8, "little")


def rc4(data: bytes, key: bytes) -> bytes:
    state = list(range(256))
    j = 0
    for i in range(256):
        j = (j + state[i] + key[i % len(key)]) & 0xFF
        state[i], state[j] = state[j], state[i]

    i = j = 0
    out = bytearray()
    for value in data:
        i = (i + 1) & 0xFF
        j = (j + state[i]) & 0xFF
        state[i], state[j] = state[j], state[i]
        stream = state[(state[i] + state[j]) & 0xFF]
        out.append(value ^ stream)
    return bytes(out)


print(rc4(enc, key))
```

输出为 `honk-mimimimimi`。因此六个答案依次是：

```text
0x402db6
strlen
0xf
0xc1de1494171d9e2f
rc4
honk-mimimimimi
```

提交全部答案后，服务读取 `flag.txt`，得到：

```text
grey{solv3d_m1_f1r5t_r3v_ch4lleng3_heh3}
```

## 方法总结

- 核心技巧：先给函数划分语义；无输入的复杂函数无需完整化简，可在稳定返回边界直接观察返回值。
- 识别信号：`S[256]` 初始化、按密钥置换以及逐字节异或是 RC4 的典型 KSA/PRGA 结构。
- 复用要点：整数转 RC4 密钥时必须确认端序；本题源码明确按低字节到高字节排列，因此使用小端字节序。
