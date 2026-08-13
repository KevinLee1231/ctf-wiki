# GreyCTF2023 MonkeyType

## 题目简述

终端打字程序维护 `idx`、`highscore` 和栈上字符数组 `buf[64]`。处理退格键 `0x7f` 时直接执行 `idx--`，没有下界检查；普通可打印字符又会写入 `buf[idx++]`。因此连续退格可把写位置移到数组之前，覆盖同一栈帧中的 `highscore`。

## 解题过程

源码的胜利条件为：

```c
if (highscore > 0xffffffff) {
    puts("grey{I_am_M0JO_J0JO!}");
}
```

`highscore` 位于 `buf` 前 72 字节。以原始终端模式连接后发送 72 个退格，再发送至少 5 个非零字符，使 64 位 `highscore` 的低若干字节变为 `0x41`：

```python
io.send(b"\x7f" * 72)
io.send(b"A" * 5)
```

此时数值已大于 `0xffffffff`，下一轮循环命中胜利分支并输出：

```text
grey{I_am_M0JO_J0JO!}
```

仓库还指出可退格 68 次后写一个字符，直接命中 `highscore` 的较高字节，同样满足条件。

## 方法总结

输入编辑逻辑也属于内存安全边界。程序虽然限制正常输入的最大长度，却没有保证光标索引非负，最终形成向前越界写。审计文本框、行编辑器和终端程序时，插入、删除、移动光标三类操作都必须维持 $0\le idx\le capacity$，不能只防止向后的溢出。
