# broken hash

## 题目简述

程序要求输入固定长度 88 的 flag。它不是对整段输入计算一次哈希，而是对每个字符分别调用自带的 SHA-1 变体，并只取摘要前 4 字节组成 `check[i]`：

```c
for (int j = 0; j < 88; j++) {
    tmp[0] = input[j];

    sha1_init(&ctx);
    sha1_update(&ctx, tmp, 1);
    sha1_final(&ctx, buf);

    check[j] =
        (buf[3] << 24)
        | (buf[2] << 16)
        | (buf[1] << 8)
        | buf[0];
}
```

随后程序从左到右比较 `check` 与内置的 `enc`，遇到第一个错误字符便退出循环。题目还用 TLS 回调执行 `IsDebuggerPresent`，再用除零异常和 SEH 隐藏真实输出路径。因此，直接单步调试时看到的控制流与正常运行并不相同。

本题有价值的核心不是求逆 SHA-1，而是把“首个错误即退出”的校验器改造成一个泄露正确前缀长度的 oracle。

## 解题过程

TLS 回调在 `main` 之前记录调试状态：

```c
void NTAPI MY_TLS_CALLBACK(
    PVOID dll_handle,
    DWORD reason,
    PVOID reserved
) {
    dbg = IsDebuggerPresent();
}
```

主函数根据 `dbg` 选择两个函数之一。正常运行时调用 `trigger_exception`；处于调试状态时改调 `hook`，从而绕过预期的异常处理路径：

```c
phook = trigger_exception;
if (dbg)
    phook = hook;

sha1(input);

for (int i = 0; i < 88; i++) {
    pass = pass && (check[i] == enc[i]);
    if (!pass)
        break;
}

__try {
    phook();
}
__except (Filter_div0(GetExceptionCode())) {
    decode(enc + 88);
    printf("%s", (char *)(enc + 88));
    return 0;
}
```

`trigger_exception` 的关键语句是 `1 / pass`。输入错误时 `pass=0`，程序触发除零异常并进入 handler；调试状态下换成不会异常的 `hook`，所以调试器里反而看不到这条真实路径。分析时可以直接 patch TLS 回调的结果，也可以先静态确定 handler，再让修改后的程序在非调试状态下运行。

比较循环第一次失败时的索引 $i$，正好等于已经匹配的前缀长度。官方预期做法是在编译结果中修改异常 handler 的输出参数，让它在原有输出末尾泄露这个栈上局部值。约定补丁后的程序把匹配长度作为最后一个原始字节输出，则可以逐位枚举：

```python
import subprocess

BINARY = "Broken_hash_patched.exe"
FLAG_LENGTH = 88
ALPHABET = range(0x21, 0x7F)


def matched_prefix_length(prefix):
    payload = prefix + b"!" * (FLAG_LENGTH - len(prefix))
    result = subprocess.run(
        [BINARY],
        input=payload + b"\n",
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL,
        check=False,
    )
    output = result.stdout

    # 完整输入正确时不会触发除零异常，而会走成功输出。
    if b"This is your flag" in output:
        return FLAG_LENGTH

    # 此解析与补丁约定一致：最后一字节是首次失败的索引。
    return output[-1]


flag = bytearray(b"moectf{")

while len(flag) < FLAG_LENGTH:
    for value in ALPHABET:
        trial = bytes(flag) + bytes([value])
        if matched_prefix_length(trial) >= len(trial):
            flag.append(value)
            print(flag.decode())
            break
    else:
        raise RuntimeError(
            f"position {len(flag)} has no printable candidate"
        )

print(flag.decode())
```

假设当前已知前缀长度为 $i$。若试探字符错误，循环会在索引 $i$ 处退出；若正确，至少会继续到 $i+1$，所以一次运行即可判断这一位。后续填充字符偶然正确只会让泄露值更大，不会导致误判。

恢复结果为：

```text
moectf{F1nd_th3_SEH_7hen_B1a5t_My_Fla9_and_Y0u_Can_Get_A_Cup_Of_Milk_Tea_From_YunZh1Jun}
```

这道题还存在更直接的弱点：每个字符被独立哈希，字符之间没有盐、位置或链式依赖。因此，只要 patch 程序导出所有可见字符对应的 4 字节值，就能建立查表映射，TLS、SEH 和前缀 oracle 都不再必要。自带 SHA-1 的内部参数经过修改，不能未经验证就用标准库的 `sha1` 结果替代。

## 方法总结

分析带 TLS 与 SEH 的校验程序时，要区分启动前回调、正常控制流和异常控制流；`IsDebuggerPresent` 可能专门让调试路径避开真实 handler。遇到逐项比较并在首错处退出的验证器，应优先考虑泄露或 patch 当前索引，把布尔校验转成前缀长度 oracle。与此同时，还要检查哈希是否按字符独立计算：若没有位置和上下文依赖，复杂的反调试外壳并不能弥补可直接查表的验证结构。
