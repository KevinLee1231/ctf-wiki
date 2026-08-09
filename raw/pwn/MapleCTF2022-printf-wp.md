# printf

## 题目简述

程序把用户输入直接作为 `printf` 的格式字符串，并在函数返回前允许反复输入。二进制启用 PIE、NX 和 Full RELRO，不能改 GOT；但栈上存在指向相邻栈槽的指针链，可以先修改“写入目标指针”，再用 `%hn` 分段覆盖保存返回地址。

## 解题过程

第一次格式串同时完成三件事：把循环控制相关字节改回可继续输入的值、泄露第 6 个参数附近的栈地址，以及泄露第 13 个参数中的 `__libc_start_main` 返回地址。后者减去本地配套 `libc-2.31.so` 的固定偏移得到 libc 基址。

由于 Full RELRO，选择覆盖栈上的返回地址。官方脚本利用一个参数指向栈上另一个指针：先用 `%hhn` 把该指针低字节改到 `saved_rip+offset`，再用 `%hn` 写入目标 16 位，最后把循环字节恢复为 `0x44`：

```python
def write_word_on_stack(offset, word):
    # %hhn 调整间接指针，%hn 写两字节，再恢复循环状态。
    ...

one_gadget = libc.address + 0xe3b01
write_word_on_stack(0, one_gadget)
write_word_on_stack(2, one_gadget >> 16)
```

目标 one-gadget 与原返回地址高位相同，只需覆盖低四字节。发送空行结束循环，函数返回到 gadget，取得 shell 并读出：

```text
maple{F0wm47_57w1ng_3xpl01t_UwU}
```

## 方法总结

格式串在 Full RELRO 下仍可攻击返回地址、函数指针或栈上指针链。规划载荷时要持续跟踪“已打印字符数”，分别对 $2^8$ 和 $2^{16}$ 取模计算 `%hhn/%hn` 的填充量。还应使用题目提供的 libc 核对 one-gadget 偏移和约束，不能把本机偏移直接套到远端。
