# Xor

## 题目简述

Windows 程序读取 44 字节输入，将每字节与 `0x24` 异或后和 `.rdata` 常量比较。IDA 最初把连续栈空间拆成 `Buffer[16]` 和多个相邻变量，这是类型恢复问题，不是一个真实的 45 字节栈溢出。

## 解题过程

反编译中可见 `fgets(Buffer, 45, stdin)`、初始化连续 45 字节，以及循环访问 `Buffer[0]` 到 `Buffer[43]`。把第一个局部变量重新定义为至少 45 字节的字符数组后，伪代码会简化为：

```c
char buffer[45] = {0};
fgets(buffer, 45, stdin);
for (int i = 0; i < 44; ++i) {
    if (((unsigned char)buffer[i] ^ 0x24) != encrypted[i])
        fail();
}
```

异或是自身的逆运算，直接对常量数组再次异或 `0x24`：

```python
encrypted = [
    0x49, 0x4B, 0x41, 0x47, 0x50, 0x42, 0x5F, 0x41,
    0x1C, 0x16, 0x46, 0x10, 0x13, 0x1C, 0x40, 0x09,
    0x42, 0x16, 0x46, 0x1C, 0x09, 0x10, 0x10, 0x42,
    0x1D, 0x09, 0x46, 0x15, 0x14, 0x14, 0x09, 0x17,
    0x16, 0x14, 0x41, 0x40, 0x40, 0x16, 0x14, 0x47,
    0x12, 0x40, 0x14, 0x59,
]

print(bytes(value ^ 0x24 for value in encrypted).decode())
```

输出：

```text
moectf{e82b478d-f2b8-44f9-b100-320edd20c6d0}
```

## 方法总结

反编译器根据栈访问猜测局部变量边界，可能把一个数组拆成多个变量。应结合初始化范围、输入长度和循环上界重设类型，再分析真正的校验；本题恢复类型后只剩固定字节 XOR。
