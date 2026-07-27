# ezjunk

## 题目简述

程序用三处重叠指令/返回地址修改干扰静态反汇编，并在构造函数中调用 `IsDebuggerPresent` 改变密钥。清理这些控制流后，核心校验由四组 64 位分组上的 XTEA 变体和八个 32 位字上的 CRC 式循环组成。

XTEA 的初始 `sum` 与 `delta` 不是普通常量，而是从两处垃圾代码地址上的机器码字节读取；反调试分支则决定四个密钥字中的两个是否异或 `0x44` 与 `0x33`。先逆转 32 轮 CRC，再对四个分组执行 XTEA 逆运算，即可恢复 32 字节 flag。

## 解题过程

### 构造函数中的垃圾指令与反调试

构造函数先把字符串逐字节异或 `0x50`，再尝试加载不存在的 `./D^3CTF.dll`。随后通过恒定分支和跳入指令中部的方式，让反汇编器把后续字节误识别为一个无意义的 `call`：

```text
004015BA  mov  rax, cs:LoadLibraryA
004015C1  call rax
004015C3  cmp  eax, 0
004015C6  jnb  loc_4015C8+1
004015C8  call 6C45A115h
```

按真实跳转目标重新定义代码后，可以看到：

```c
LoadLibraryA(decoded_name);
if (IsDebuggerPresent()) {
    key_area[100] ^= 0x44;
    key_area[103] ^= 0x33;
} else {
    key_area[101] ^= 0x44;
    key_area[102] ^= 0x33;
}
```

正常运行应走 `else`，对应最终密钥：

```c
uint32_t key[4] = {
    0x5454,
    0x4646 ^ 0x44,
    0x4444 ^ 0x33,
    0x5e5e
};
```

动态调试时可让 `IsDebuggerPresent` 返回 0，或在确认控制流后直接采用正常分支的密钥。

### `main` 中修改返回地址的垃圾代码

第二处代码先 `call loc_401A27`，被调函数随后执行：

```text
add [rsp], 0x12
...
retn
```

它把栈顶返回地址增加 `0x12`，所以 `retn` 并不会回到 `call` 后的线性地址。`cmp`、条件跳转与夹在其中的 `0xe9` 字节主要用于诱导错误反汇编。沿修改后的真实返回地址重新建函数后，`main` 的有效逻辑为：

1. 异或 `0x50` 输出提示；
2. 用 `gets` 读入候选；
3. 每 8 字节调用一次自定义 XTEA，共处理四组；
4. 检查后续 32 位变换结果；
5. 成功时输出异或编码的提示。

### 析构函数中的栈指针扰动

第三处位于析构函数。辅助块先弹出返回地址，将其加一，再通过交换栈顶和 `rsp` 恢复栈结构：

```text
pop  rax
add  rax, 1
push rax
mov  rax, rsp
xchg rax, [rax]
pop  rsp
mov  [rsp], rax
retn
```

因此返回时会跳过一个伪造的 `0xe8` 字节，落到真正的短跳转。按这条路径清理后，可以看到八个目标常量和逐字执行 32 轮的 CRC 式校验：

```c
if (word & 0x80000000) {
    word = (word << 1) ^ 0x84a6972f;
} else {
    word <<= 1;
}
```

目标常量为：

```c
{
    0xb6ddb3a9, 0x36162c23, 0x1889fabf, 0x6ce4e73b,
    0x0a5af8fc, 0x21ff8415, 0x44859557, 0x2dc227b7
}
```

### 逆转 CRC 式循环

多项式 `0x84a6972f` 的最低位为 1，因此可由输出最低位判断正向步骤使用了哪个分支：

- 输出最低位为 0：正向输入最高位为 0，逆运算是右移一位；
- 输出最低位为 1：正向输入最高位为 1，先异或多项式、右移一位，再补回最高位。

逆运算为：

```c
if (word & 1) {
    word = ((word ^ 0x84a6972f) >> 1) | 0x80000000;
} else {
    word >>= 1;
}
```

每个目标字执行 32 次即可恢复 XTEA 阶段的密文。

### XTEA 变体与完整求解器

从垃圾代码地址取出的参数为：

```text
initial sum = 0xe8017300
delta       = 0xff58f981
```

轮函数不是标准 XTEA：两半的移位常量分别为 `(4, 5)` 与 `(5, 6)`，并额外异或 `0x44`、`0x33`。下面的完整 C 程序先逆 CRC，再解密四个 64 位分组：

```c
#include <stdint.h>
#include <stdio.h>

static void xtea_decrypt(
    uint32_t block[2],
    uint32_t initial_sum,
    uint32_t delta,
    const uint32_t key[4])
{
    uint32_t v0 = block[0];
    uint32_t v1 = block[1];
    uint32_t sum = initial_sum - delta * 32U;

    for (unsigned int round = 0; round < 32; ++round) {
        sum += delta;
        v1 -= (
            (v0 + ((v0 << 5) ^ (v0 >> 6)))
            ^ (key[(sum >> 11) & 3] + sum)
            ^ 0x33
        );
        v0 -= (
            (v1 + ((v1 << 4) ^ (v1 >> 5)))
            ^ (key[sum & 3] + sum)
            ^ 0x44
        );
    }

    block[0] = v0;
    block[1] = v1;
}

static uint32_t reverse_crc_rounds(uint32_t value)
{
    for (unsigned int round = 0; round < 32; ++round) {
        if (value & 1) {
            value =
                ((value ^ 0x84a6972fU) >> 1)
                | 0x80000000U;
        } else {
            value >>= 1;
        }
    }
    return value;
}

int main(void)
{
    uint32_t data[8] = {
        0xb6ddb3a9, 0x36162c23, 0x1889fabf, 0x6ce4e73b,
        0x0a5af8fc, 0x21ff8415, 0x44859557, 0x2dc227b7
    };
    const uint32_t key[4] = {
        0x5454,
        0x4646 ^ 0x44,
        0x4444 ^ 0x33,
        0x5e5e
    };

    for (unsigned int i = 0; i < 8; ++i) {
        data[i] = reverse_crc_rounds(data[i]);
    }

    for (unsigned int i = 0; i < 8; i += 2) {
        xtea_decrypt(
            &data[i],
            0xe8017300U,
            0xff58f981U,
            key
        );
    }

    fwrite(data, 1, sizeof(data), stdout);
    fputc('\n', stdout);
    return 0;
}
```

输出：

```text
d3ctf{ea3yjunk_c0d3_4nd_ea5y_re}
```

## 方法总结

三处垃圾代码的共同点都是破坏线性反汇编：跳入指令中部、修改返回地址以及手工重排栈顶。处理时应以实际控制流和栈变化为准，不要把反汇编器生成的所有边都视为有效。

有效校验可分成“CRC 式可逆位移”和“XTEA 变体”两层。密钥、`sum`、`delta` 和额外异或常量都藏在垃圾代码或反调试分支里，不能直接套标准 XTEA。最终求解器使用无符号 32 位类型，让加减法自然按模 $2^{32}$ 回绕，避免有符号溢出引入未定义行为。
