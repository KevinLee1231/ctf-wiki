# 16bit

## 题目简述

附件是一个 686 字节的标准 DOS MZ 16 位可执行文件，入口代码位于文件偏移 `0x200`。64 位 Windows 不能直接运行这种 DOS 程序，但它仍可由 DOSBox 执行，也可以按 16 位 x86 指令静态反汇编。

程序没有等待输入，而是对内置的 46 字节数据进行两轮解码，再通过 DOS 中断 `int 21h` 的 `AH=09h` 功能输出以 `$` 结尾的字符串。

## 解题过程

直接把 `16bit.exe` 放入 DOSBox，进入挂载目录后执行 `16bit`，程序会打印最终 flag。若不依赖模拟器，入口处的两段循环也很短：

```asm
mov cx, 0x17
loop1:
    mov al, [si + 0x0a]
    sub al, 0x09
    xor al, 0x0e
    mov [di + 0x38], al
    inc si
    inc di
    loop loop1

mov cx, 0x17
loop2:
    mov al, [si + 0x0a]
    xor al, 0x0e
    sub al, 0x09
    mov [di + 0x38], al
    inc si
    inc di
    loop loop2
```

两轮都处理 `0x17 = 23` 字节，但减法与异或的顺序相反。文件中的编码数据为：

```text
47 7f 52 78 6c 74 7e 72 47 47 73 5a 84 5a 43 85
46 5a 83 6f 46 5a 6c 33 30 73 32 75 66 37 61 66
33 30 78 66 40 35 61 4e 64 34 65 32 33 88
```

按汇编顺序模拟即可：

```python
encoded = bytes.fromhex(
    "47 7f 52 78 6c 74 7e 72 47 47 73 5a 84 5a 43 85 "
    "46 5a 83 6f 46 5a 6c 33 30 73 32 75 66 37 61 66 "
    "33 30 78 66 40 35 61 4e 64 34 65 32 33 88"
)

first = bytes((((value - 9) & 0xFF) ^ 0x0E) for value in encoded[:23])
second = bytes((((value ^ 0x0E) - 9) & 0xFF) for value in encoded[23:])
print((first + second).decode())
```

输出为：

```text
0xGame{g00d_u_4r3_th3_m45t3r_0f_45m_E2f7a1b34}
```

## 方法总结

16 位 DOS EXE 无法在现代 64 位 Windows 中直接运行，不代表只能依赖截图或在线工具。可用 DOSBox 动态执行，也可从 MZ 入口开始静态分析。本题的核心只有两个 23 次循环，必须保留“第一轮先减后异或、第二轮先异或后减”的顺序，否则解码结果会出错。
