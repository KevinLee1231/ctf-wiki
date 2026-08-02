# TJCTF2022 woodchipper

## 题目简述

程序实现一台 16 位自定义虚拟机。指令字编码操作数个数、寻址方式和寄存器，而低 5 位 opcode 还要经过一棵深度为 5 的二叉树才能映射到具体函数。随附字节码会根据自身 `prog.bin` 的字节频率建立 Huffman 树，读取用户提交的压缩数据，校验长度和 CRC16，再把解压结果当作第二段 VM 程序执行。

## 解题过程

先从 `machine.c` 确认取指与寻址规则，并从 `instructions.c` 的 `init_tree` 恢复 32 个 opcode 到 `add`、`mov`、`sys` 等函数的映射。该映射可画成决策树，便于核对低 5 位的逐位分支：

![woodchipper 虚拟机低五位 opcode 到指令函数的二叉决策树](TJCTF2022-woodchipper-wp/vm-instruction-tree.png)

然后用题目汇编器生成调试版 `prog-dbg.bin`。它严格执行正式程序中的直方图和 Huffman 建树逻辑，再通过 `write(2, ...)` 导出根索引及节点数组。每个节点是四个 16 位值 `(symbol, frequency, left, right)`；从根递归遍历，左边追加 0、右边追加 1，即可得到每个字节的 Huffman 编码。

最终载荷不是直接提交 flag 文本，而是一段读取 `flag.txt` 并写到标准输出的 VM 程序。用 `asm.py flag` 得到其字节码，偶数长度补齐后计算 CRC16，再按刚恢复的码表压缩。文件格式为：

```python
solution = p16(len(stage2)) + crc16_le + huffman_bits_packed_as_le16
```

虚拟机验证并解压后执行第二段程序，从而输出 `tjctf{s0_do_w3_c4ll_y0u_an_arb0r1st_n0w??}`。

## 方法总结

本题包含两层解释：外层是自定义指令集，内层是由输入数据频率动态生成的 Huffman 编码。稳妥做法是先用官方 VM 的调试变体导出建树结果，避免自行实现时在相同频率排序、节点编号或位序上产生偏差；再复用官方汇编器生成真正的读 flag 程序。每一层都应有独立的中间产物和校验，而不是直接猜最终压缩流。
