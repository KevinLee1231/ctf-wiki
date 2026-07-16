# 花鸟风月

## 题目简述

附件是一个 32 位 Windows PE。程序先校验用户输入的 32 位十六进制 key，再读取 flag，依次执行循环 XOR 和一段变形 Base64，最后与程序内加密保存的目标串比较。二进制同时用互斥的 `jz`/`jnz`、`call $+5` 配合 `retn` 等花指令破坏 IDA 的控制流恢复，并对提示字符串做了运行时解密，不能直接相信初始反编译结果。

## 解题过程

### 修复外层控制流与字符串

入口附近在 `0x4012EE` 和 `0x4012F0` 连续出现目标相同的 `jnz`、`jz`。无论零标志为何，程序都会跳到 `0x4012F2`，但该位置的垃圾字节会令 IDA 从错误边界解码。将 `0x4012F2` 处的花指令 NOP 掉并重新分析，即可恢复主函数。

程序中的 `unk_422054` 等区域看似是无意义字节，但运行时可以正常打印提示。动态跟踪可确认 `sub_401000` 是字符串解密函数，`sub_401670` 是打印包装函数。整理变量后，主校验链可以概括为：

```c
read_hex_key(&key);
if (check_key(key) != 0xA745DA5D) {
    print("Key is incorrect!");
    return 0;
}

read_flag(flag);
xor_transform(flag, key, 44);
modified_base64(flag, encoded, 44);

if (strncmp(encoded, expected, 60) == 0)
    print("Congratulations!");
```

### 求出第一阶段 key

key 校验以 `0x81428113` 为初值，循环十次执行 32 位乘加，再异或输入 key 的循环左移结果：

```c
hash = 0x81428113;
for (i = 0; i < 10; ++i) {
    hash = 0x01421043 * hash + 0x43298815;
    hash ^= rol32(key, 13 * i % 32);
}
```

目标值是 `0xA745DA5D`。下面的穷举程序按 `uint32_t` 自然截断到 32 位，并避免移位量为零时出现未定义行为：

```c
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>

static uint32_t rol32(uint32_t value, unsigned int count) {
    count &= 31;
    return count == 0 ? value : (value << count) | (value >> (32 - count));
}

int main(void) {
    for (uint64_t candidate = 0; candidate <= UINT32_MAX; ++candidate) {
        uint32_t key = (uint32_t)candidate;
        uint32_t hash = 0x81428113;

        for (unsigned int i = 0; i < 10; ++i) {
            hash = 0x01421043 * hash + 0x43298815;
            hash ^= rol32(key, 13 * i % 32);
        }

        if (hash == 0xA745DA5D) {
            printf("0x%08" PRIX32 "\n", key);
            return 0;
        }
    }
    return 1;
}
```

得到第一阶段输入 key：

```text
0x98BF3B77
```

### 还原目标串与真实 XOR 字节流

程序内的目标数据先按逆序恢复并异或 `0x99`，可得到被打乱的 Base64 字符串：

```text
MAPDc3GtPQ34vM3pNBb1PWGxccHtvxH1NVa9JPTtacS57IilN5uxJSW5Iwm=
```

变形编码每四个字符按 `c1, c3, c2, c4` 输出，即中间两个字符交换。逐组换回标准顺序后得到：

```text
MPADcG3tP3Q4v3MpNbB1PGWxcHctvHx1NaV9JTPtaSc57iIlNu5xJWS5Imw=
```

静态伪代码看起来把第一阶段输入 key 直接传给 XOR 函数，但该函数内部还有一层 `call $+5`、真假分支和两个 `retn` 组成的控制流欺骗。其关键片段为：

```asm
xor     eax, eax
xor     eax, 1
call    $+5
xor     eax, 1
cmp     eax, 1
jz      short real_path
retn
```

`call $+5` 会把下一条指令地址压栈；后续 `retn` 并非真正返回调用者，而是回到该地址继续执行。动态输入一串字符 `1`（字节 `0x31`），观察变换后周期性出现 `31 B9 75 20`，逐字节异或即可反推出真正的循环字节流为 `00 88 44 11`。

最终按“修正 Base64 字符顺序 → Base64 解码 → 循环 XOR”的逆序恢复：

```python
import base64

scrambled = "MAPDc3GtPQ34vM3pNBb1PWGxccHtvxH1NVa9JPTtacS57IilN5uxJSW5Iwm="

standard = "".join(
    scrambled[i] + scrambled[i + 2] + scrambled[i + 1] + scrambled[i + 3]
    for i in range(0, len(scrambled), 4)
)

encrypted = base64.b64decode(standard)
xor_key = bytes.fromhex("00 88 44 11")
flag = bytes(value ^ xor_key[i % 4] for i, value in enumerate(encrypted))
print(flag.decode())
```

输出为：

```text
0xGame{e8778581-e94f-48d5-943e-69ff46f54d1f}
```

因此 flag 为 `0xGame{e8778581-e94f-48d5-943e-69ff46f54d1f}`。

## 方法总结

本题的主线是先修控制流，再逆两层数据变换。遇到同一目标的互补条件跳转、跳入指令中间、`call $+5` 后接 `retn`，应优先怀疑花指令，而不是继续依赖错误伪代码。修复外层反汇编后，还要用动态输入验证每个变换函数的真实效果：已知输入与输出的逐字节差分可以直接恢复循环 XOR 字节流；识别 Base64 时则不能只看字符表，还要检查四字符输出顺序是否被置换。
