# cmsc430

## 题目简述

附件只有一个带调试符号的 64 位 ELF。题目说明它由 CMSC430 课程中的手写 Racket 编译器生成。程序逐字节读取输入并返回 `#t` 或 `#f`；需要理解该编译器对整数的运行时编码，再还原每个比较常量对应的字符。

## 解题过程

`file` 和符号表表明二进制未剥离，关键函数名包括 `entry`、`read_byte` 和 `val_wrap_int`。反汇编 `read_byte` 可以看到它调用 `getc`，随后把读到的字节交给 `val_wrap_int`：

```asm
call   getc
movsx  rax, BYTE PTR [rbp-0x1]
mov    rdi, rax
call   val_wrap_int
```

`val_wrap_int` 的有效逻辑只有一条加法：

```asm
mov    rax, rdi
add    rax, rax
ret
```

也就是说，编译器把整数 $x$ 表示为 $2x$，最低位留给运行时类型标签。`entry` 中每次调用 `read_byte` 后都会加载一个偶数常量并比较，例如：

```asm
call   read_byte
push   rax
mov    eax, 0xaa
pop    r8
cmp    r8, rax
```

因此首字节满足：

$$
2c=0xaa,\qquad c=0x55=\texttt{U}
$$

后续常量 `0x9a`、`0x88`、`0x86` 分别除以 2 后得到 `M`、`D`、`C`。可以手工抄录全部立即数，也可以按“每次 `read_byte` 调用后的第一个 `mov eax, imm`”自动提取：

```python
import re
import subprocess

disassembly = subprocess.check_output(
    [
        "objdump",
        "-d",
        "-Mintel",
        "--disassemble=entry",
        "./chal",
    ],
    text=True,
)

blocks = re.split(
    r"call\s+[0-9a-f]+\s+<read_byte>",
    disassembly,
)[1:]

encoded = []
for block in blocks:
    match = re.search(r"mov\s+eax,0x([0-9a-f]+)", block)
    if match:
        encoded.append(int(match.group(1), 16))

assert all(value % 2 == 0 for value in encoded)
flag = bytes(value // 2 for value in encoded)
print(flag.decode())
```

输出：

```text
UMDCTF{shout_out_to_jose}
```

把结果交给程序验证：

```text
$ printf 'UMDCTF{shout_out_to_jose}\n' | ./chal
#t
```

错误输入则输出 `#f`。反汇编中反复出现的 `3`、`7` 不是字符常量，而是该运行时使用的布尔标签，提取时不能把它们混入结果。

## 方法总结

这题没有复杂加密，障碍是编译器的 tagged value 表示。沿 `read_byte` 跟到 `val_wrap_int` 后，可以确定输入字符在比较前被乘以 2，于是把 `entry` 中每个期望立即数除以 2 即可恢复 flag。带符号函数名已经给出清晰入口，先确认数据表示比逐块分析重复的条件分支更高效。
