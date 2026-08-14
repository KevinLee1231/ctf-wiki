# bi0sCTF 2024 - beehive

## 题目简述

附件是一个 eBPF ELF 对象。程序挂在 `raw_tracepoint/sys_enter`，只处理自定义 syscall 编号 `0x31337`，从第一个参数读取字符串，把每个字符的 8 个比特反转后与内置整数数组比较。题目要求恢复原始 key 并包在 `bi0sctf{...}` 中。

仓库 README 将 syscall 写成 `0x1337`，但源码中的实际常量是 `0x31337`；静态分析或复现时应以目标对象和源码为准。

## 解题过程

### 定位 eBPF 入口与输入

从 ELF section 或反编译结果可见主函数位于 `raw_tracepoint/sys_enter`。它从 tracepoint 上下文取得 syscall id，并在编号不等于 `0x31337` 时立即返回。命中后，从 `pt_regs.rdi` 读取用户态字符串，最多读入 32 字节。

程序中的核心变换为：

```c
for (int j = 0; j < 8; j++) {
    padded_binary |= (temp & (1 << j))
        ? (1 << (8 - j - 1)) : 0;
}
```

这不是按位取反，而是把一个字节的位序倒转。例如二进制 `01101000` 会变成 `00010110`。

### 逆转硬编码数组

比较数组共有 29 项：

```text
86 ae ce ec fa 2c 76 f6 2e 16 cc 4e fa ae ce cc
4e 76 2c b6 a6 02 46 96 0c ce 74 96 76
```

位反转是对合变换，即 $\operatorname{rev}_8(\operatorname{rev}_8(x))=x$，所以对每个数组元素再反转一次即可恢复输入：

```python
expected = [
    86, 174, 206, 236, 250, 44, 118, 246, 46, 22,
    204, 78, 250, 174, 206, 204, 78, 118, 44, 182,
    166, 2, 70, 150, 12, 206, 116, 150, 118,
]

def reverse_bits8(value):
    return int(f"{value:08b}"[::-1], 2)

key = bytes(reverse_bits8(value) for value in expected)
print(key.decode())
```

结果是一段用户名/邮箱形式的可打印字符串。将其作为花括号内内容即可得到 flag。

源码还有一个长度校验缺陷：若输入短于 29 字节，循环结束后没有验证 `reversed_ctr == reversed_keys_length`，甚至空字符串也可能保留 `is_correct = 1`。不过这只能让日志打印“Key is correct”，并不能替代题目要求的 key 恢复；上面的逆变换给出了确定的完整答案。

## 方法总结

识别 eBPF 挂载点后，真正需要逆向的只有一个逐字节位序反转。要区分“bit reverse”和“bitwise NOT”：前者改变位的位置，后者才是逐位取反。位序反转执行两次回到原值，因此无需暴力枚举。对于文档与源码不一致的 syscall 常量，应以实际对象中的比较立即数为准。
