# Calamity_Fortune

## 题目简述

附件是一个经过 strip 的 64 位 Windows PE。表面流程让用户猜 1～100 的随机数：猜错会由 `YourGift()` 输出一个伪 flag，猜中才调用 `MessageBoxA()`。真正的入口藏在 `main()` 之前：带 `constructor` 属性的函数会安装 x64 inline hook，把 `MessageBoxA` 的入口改写到 `FlagMessageBoxA`，44 字节 flag 的校验链就在这个替代函数中。

决定程序结构的代码可收敛为：

```c
static void __attribute__((constructor)) auto_install_hook(void) {
    InstallInlineHook_MessageBoxA();
}

int main(void) {
    srand((unsigned)time(NULL));
    int secret = (rand() % 100) + 1;
    int guess = 0;
    scanf("%d", &guess);

    if (guess == secret) {
        MessageBoxA(NULL, "You guessed right! Is it really right?",
                    "Result", MB_OK);
    } else {
        YourGift();
    }
}
```

Hook 会把目标函数开头改写成 `mov rax, imm64; jmp rax`。跳转目标是 `FlagMessageBoxA`，所以不能只沿着 `main()` 的显式调用关系判断哪个分支含有真 flag。

## 解题过程

### 1. 排除猜错分支的伪 flag

`YourGift()` 中的 46 字节数组只与循环密钥 `Calamity` 异或，解出：

```text
0xGame{this_is_a_fake_flag_please_keep_trying}
```

这条路径没有进入 60 字节目标数组的比较，因此只是诱饵。要到达真实校验，可以在调试器中修改猜数分支，也可以直接分析程序启动时安装的 Hook。

### 2. 确认真正入口是 inline hook

`InstallInlineHook_MessageBoxA()` 先解码字符串 `MessageBoxA`，再通过 `GetProcAddress` 取得函数地址。`HookInline_x64()` 保存入口处 12 字节到 trampoline，随后写入以下等价指令：

```asm
mov rax, FlagMessageBoxA
jmp rax
```

其机器码骨架为 `48 b8 <8-byte address> ff e0`。代码还调用了 `VirtualProtect` 和 `FlushInstructionCache`，说明修改的是函数正文而不是 IAT 表项。因此 `main()` 中看似普通的 `MessageBoxA()` 最终会跳入 `FlagMessageBoxA()`。

### 3. 梳理真实校验链

`FlagMessageBoxA()` 要求输入长度严格等于 44，并依次执行：

1. 按小端序把 44 字节拆成 11 个 `uint32_t`。
2. 使用标准 XXTEA 加密，`DELTA = 0x9E3779B9`。
3. 四个 32 位密钥字为 `0x616C6143, 0x7974696D, 0x726F465F, 0x656E7574`；按小端拼接后是 `Calamity_Fortune`。
4. 把 11 个加密字重新转成 44 字节，再做标准 Base64 编码。
5. 44 字节经 Base64 编码后得到 $4\lceil44/3\rceil=60$ 字节；以 `srand(101)` 初始化后执行 Fisher–Yates 原地打乱。
6. 每个字节异或 `0x45`，最后与内置的 `trueFlag[60]` 比较。

原题解将第二步之后写成“Base64 解码”，但程序实际调用的是 `base64_encode()`。求解时必须严格逆转程序的正向链，即先撤销异或与置换，再 Base64 解码，最后做 XXTEA 解密。

### 4. 逆变换恢复 flag

这里还存在一个平台细节：附件是 Windows PE，`srand()`/`rand()` 使用 MSVCRT 实现。Python `random` 和 glibc `rand()` 都不会产生相同序列；MSVCRT 每轮按 `state = state * 214013 + 2531011` 更新，并返回状态的高 15 位。

下面的完整脚本依次撤销异或、MSVCRT Fisher–Yates 置换、Base64 和 XXTEA：

```python
import base64
import struct


MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
TRUE_FLAG = bytes([
    116, 120, 7, 43, 115, 48, 13, 40, 11, 6,
    115, 53, 16, 0, 45, 36, 115, 47, 125, 114,
    6, 52, 28, 20, 51, 40, 54, 118, 118, 19,
    113, 10, 42, 116, 4, 47, 36, 43, 35, 14,
    16, 115, 61, 3, 3, 63, 55, 47, 125, 45,
    119, 18, 54, 124, 6, 119, 4, 49, 35, 55,
])
KEY = [0x616C6143, 0x7974696D, 0x726F465F, 0x656E7574]


def msvcrt_rand_sequence(seed):
    state = seed & MASK
    while True:
        state = (state * 214013 + 2531011) & MASK
        yield (state >> 16) & 0x7FFF


def undo_shuffle(data):
    values = bytearray(data)
    rand = msvcrt_rand_sequence(101)
    swaps = []

    for index in range(59, 0, -1):
        swaps.append((index, next(rand) % (index + 1)))

    for index, target in reversed(swaps):
        values[index], values[target] = values[target], values[index]

    return bytes(values)


def mx(z, y, total, key, position, selector):
    return (
        (
            ((z >> 5) ^ ((y << 2) & MASK))
            + ((y >> 3) ^ ((z << 4) & MASK))
        )
        ^ (
            (total ^ y)
            + (key[(position & 3) ^ selector] ^ z)
        )
    ) & MASK


def xxtea_decrypt(words, key):
    values = list(words)
    count = len(values)
    rounds = 6 + 52 // count
    total = (rounds * DELTA) & MASK
    y = values[0]

    while rounds:
        selector = (total >> 2) & 3

        for position in range(count - 1, 0, -1):
            z = values[position - 1]
            values[position] = (
                values[position]
                - mx(z, y, total, key, position, selector)
            ) & MASK
            y = values[position]

        z = values[count - 1]
        values[0] = (
            values[0]
            - mx(z, y, total, key, 0, selector)
        ) & MASK
        y = values[0]
        total = (total - DELTA) & MASK
        rounds -= 1

    return values


shuffled_base64 = bytes(value ^ 0x45 for value in TRUE_FLAG)
encoded = undo_shuffle(shuffled_base64)
encrypted = base64.b64decode(encoded, validate=True)

assert len(encrypted) == 44
encrypted_words = struct.unpack("<11I", encrypted)
plain_words = xxtea_decrypt(encrypted_words, KEY)
flag = b"".join(struct.pack("<I", word) for word in plain_words)

print(encoded.decode())
print(flag.decode())
```

输出为：

```text
3C46snFsrVrpN8uqa2AAxtKmQhB6mE17ohz2YWCj6O8jvFfCaUj3nH961fU=
0xGame{f279c1e7-8b0d-4a3b-9c6f-5e4d2a1b0c89}
```

## 方法总结

本题用“猜错也给 flag”的表象隐藏伪答案，再借构造函数中的 inline hook 把正确分支转到另一套校验。分析 Windows 程序时，若同时看到 `VirtualProtect`、`FlushInstructionCache` 和 `mov rax, imm64; jmp rax`，应优先检查运行时代码改写，而不能只沿 `main()` 的静态控制流追踪。

逆向混合变换必须严格按相反顺序撤销：先异或，再逆置换，然后 Base64 解码，最后 XXTEA 解密。固定随机种子也不代表跨平台序列相同；复现 `rand()` 时应以目标程序实际使用的运行库为准。本题的决定性细节就是 MSVCRT 随机序列。
