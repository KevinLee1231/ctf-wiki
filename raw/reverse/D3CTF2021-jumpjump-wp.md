# jumpjump

## 题目简述

本题是 x86_64 静态链接 ELF 逆向签到题，未加壳、无符号表，主要考点是识别 C 标准库中的 `setjmp` / `longjmp` 异常式跳转。程序先用一次 `setjmp`/`longjmp` 检查输入长度并对输入逐字节异或，再用第二组跳转在逐位校验失败时返回 `main`。

题目附件包含静态链接 ELF、源码和 libc 信息，程序用 `setjmp/longjmp` 把长度检查与逐字节校验拆成跳转流程。真实校验是 `((buf[i] + 4) ^ 0x33) == magic[i]`，而输入在长度检查阶段已经先异或 `0x57`，所以提取 `magic` 数组后即可反推 flag。

## 解题过程

题目源码见 [`Reverier-Xu/D3CTF-jumpjump`](https://github.com/Reverier-Xu/D3CTF-jumpjump)。下文保留了源码中的 `setjmp/longjmp` 控制流、`magic` 数组和反解表达式。

本题考查点是C语言标准库中setjmp 与longjmp 的逆向识别, 作为逆向签到题目, 难度很低.

题目程序是x86_64 架构的elf 文件, 静态链接, 无符号表, 未加壳.

前置知识

`setjmp` 库类似跨函数 `goto`，会利用 `jmp_buf` 保存一个函数状态。第一次调用 `setjmp` 时返回值恒为 0；后续通过 `longjmp` 跳回该位置时，`setjmp` 的返回值变成 `longjmp` 的第二个参数。

题解

将程序拖入 IDA 后，通过字符串窗口可以快速定位到主程序 `sub_40197C`。程序中的静态字符串还暴露了编译时静态链接的 libc 版本和发行版包版本，为 `glibc 2.33`。拿到对应 libc 后，可以用 rizzo 插件还原一部分符号信息；如果熟悉 C 异常式跳转或 `setjmp` 底层实现，也可以手动识别。
简单识别一下main函数里的各个函数:

可以发现程序对输入的flag进行了两次check , 一次是sub_40189D , 另一次利用setjmp设置跳转标志,
然后调用sub_40191E 进行验证. 进入sub_40189D , 这里也通过setjmp 设置了自动跳转:

```c
bool check1(char *input) {
    int jmp_flag = setjmp(jmp_buf_check1);
    if (!jmp_flag) {
        check_len_and_xor(input);
    }
    return jmp_flag == 36;
}
```

查看sub_401825 函数:

```c
void check_len_and_xor(char *input) {
    int input_len = strlen(input);
    if (input_len != 36) {
        longjmp(jmp_buf_check1, input_len);
    }
    for (int i = 0; i < 36; ++i) {
        input[i] ^= 0x57;
    }
    longjmp(jmp_buf_check1, 36);
}
```

这两个函数结合起来就是测试输入长度是否为 36：长度正确返回 true，否则返回 false；同时在长度正确时对输入数组逐字节异或 `0x57`。

接着看第二个check , 在main函数的过程中是这样的:

```c
jmp_flag_main = setjmp(jmp_buf_main);
    if ( !jmp_flag_main )
      check2(input, 36LL);
    if ( jmp_flag_main == 1 )
    {
      printf("Sorry.\n");
      exit(0LL);
    }
printf("Good!\n");
exit(0);
```

`check2` 及其子函数通过跳转 `jmp_buf_main` 把检测结果传回 `main`：传递值为 1 时表示错误；传递值既不是默认 0，也不是 1 时表示正确。

check2 函数很简单:

```c
void check2(char *buf, int n) {
    for (int i = 0; i < n; ++i) {
        if (((buf[i] + 4) ^ 0x33) != magic[i]) {
            longjmp(jmp_buf_main, 1);
        }
    }
    longjmp(jmp_buf_main, 2);
}
```

通过sub_4018E0 对每一位进行检测, 如果不对就立即跳回main函数并输出.

主要的逻辑还是异或运算.

解密脚本十分简单, 提取出dword_4CC100 数组:

```python
magic = [
    0x09, 0x0B, 0x06, 0x5A, 0x5B, 0x0A, 0x54, 0x05, 0x4D,
    0x57, 0x56, 0x54, 0x0B, 0x4D, 0x54, 0x09, 0x55, 0x40,
    0x4D, 0x09, 0x06, 0x59, 0x0B, 0x4D, 0x55, 0x54, 0x58,
    0x57, 0x5B, 0x09, 0x0B, 0x40, 0x05, 0x0A, 0x05, 0x09,
]

flag = "".join(chr(((x ^ 0x33) - 4) ^ 0x57) for x in magic)
print(flag)
```

输出：

```text
acf23b4e-764c-4a58-af1c-54073ac8ebea
```

由于算法很简单，也可以动态调试观察跳转逻辑，或用 pintools 插桩自动记录比较值。

## 方法总结

- 核心技巧：识别 `setjmp` 第一次返回 0、`longjmp` 后返回指定值的控制流语义，把跨函数跳转还原成普通条件分支。
- 识别信号：静态链接 ELF 中出现保存上下文的全局 `jmp_buf`，校验函数没有正常返回而是调用 `longjmp`，`main` 根据跳转返回值输出成功或失败。
- 复用要点：这类题不要只按反编译出来的异常控制流硬读；先把 `setjmp/longjmp` 语义手工重构，再提取常量数组和逐字节表达式反解。

