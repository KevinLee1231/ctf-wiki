# vm-v2

## 题目简述

`vm-v2` 修复了初版 `vm` 在 `data.txt` 中直接保存明文 flag 的问题，CPU 和 `prog.txt` 不变。虚拟机指令宽 12 位，高 4 位是 opcode、低 8 位是立即数或地址；数据单元为 8 位，另有数据栈和调用栈。目标是让地址 `0x87`（135）的结果值变为 2。

## 解题过程

沿 SystemVerilog 的 case 分支和多路选择器可恢复指令集：`0..6` 是算术/逻辑，`7` 为 pop，`8` 为 jump，`9` 为 call，`a` 为 ret，`b` 为条件跳转，`c` 为 push immediate，`d/e` 为 load/store，`f` 为 halt。把 `prog.txt` 按此格式反汇编后，checker 的主体是一套缩小到 128 字节状态的 RC4 类算法。

初始化阶段令 $S[i]=i$，用地址 `0xf8..0xff` 的 8 字节 key 执行 KSA；输出阶段执行 PRGA，并把密钥流与地址 `0x8c` 起的候选 flag 异或，再与地址 `0xbc` 起的目标密文比较。循环完成且全部相等时写入 `data[0x87]=2`。

实现一个只需约百行的解释器即可运行原始 ROM。也可以直接按反汇编所得算法重写 RC4 变体，用 `data.txt` 中的 key 和密文逆运算：

```python
for i in range(128):
    j = (j + S[i] + key[i & 7]) & 0x7f
    S[i], S[j] = S[j], S[i]
# PRGA 同样对索引取模 128，再与 data[0xbc:] 异或。
```

恢复结果为：

```text
maple{the_4lag_shOUld_Not_be_put_1N_initial_RAM}
```

## 方法总结

硬件描述语言中的 VM 仍应按 CPU 的常规边界分析：指令切片、数据栈、调用栈、PC 更新和零标志。恢复 opcode 后，复杂度迅速降为普通 RC4 变体。解释器的价值在于逐条复核地址和栈语义；仅凭算法相似性重写时，最容易漏掉状态长度 128 和索引掩码 `0x7f`。
