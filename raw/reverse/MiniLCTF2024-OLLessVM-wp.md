# miniLCTF 2024 OLLessVM Writeup

## 题目简述

题名提示“不是 OLLVM”。程序实际采用基于基本块的控制流混淆：分发逻辑插入大量 `cmp/jnz`，用 EAX 作为状态，且频繁以 `pushf/popf` 保存和恢复标志寄存器。恢复真实校验块后，输入字节只与固定异或表和下标计数器异或。

## 解题过程

### 识别并去除块分发

混淆函数序言固定为：

```asm
push rax
pushf
mov eax, 0
```

每个真实块前恢复 EAX 与标志，块尾再次保存它们，再进入比较分发。可先用 Capstone 枚举真实块与所有“保存寄存器”的尾部位置，从这些位置开始模拟分发，记录实际后继块，最后把尾部 patch 成直接 `jmp successor`。开头跳入第一个真实块，其余分发指令 NOP 后，反编译器即可恢复线性控制流。官方提供的 [DeCatraz 脚本](https://github.com/doct0r3/DeCatraz/blob/master/main.py)实现了这一思路；关键原理是模拟分发状态并重连真实块，而不是把脚本当黑盒运行。

动态分析也可以在输出缓冲区下硬件写断点，直接看到核心循环：

```c
out[i] = input[i] ^ xor_table[i] ^ i;
```

程序随后把 `out` 与 `answer` 比较，因此关系可直接反解：

$$input_i=i\oplus xor\_table_i\oplus answer_i.$$

### 直接恢复输入

```python
xort = bytes.fromhex(
    "9199417B79814BCBA9EC2E02CB94E526910BA60F2881A160D"
    "1525FC47AAD4FFFE299D57A286EC037F570E6460707A2F54"
    "B393A97328EB0E7BBE8C7D2B7087B628ED06FBF369F0000"
    "A0F4A55946024602"
)
ans = bytes.fromhex(
    "FCF12D1131C7198ADABC147C98EADB65F729D04348FC8428"
    "F92923AC59CD51E0C2B8F7590C4BE610DD59CC6D2B2A8CD"
    "A7B0808A406BB86D083D1FDE98B35455D514CD172F6B8E69"
    "EE2B72D7525712B4B864587A1C947C55A165E1AD1179D186"
    "E3FD275E9E35156C206046D1A50657DFDA912"
)

plain = bytes(i ^ x ^ y for i, (x, y) in enumerate(zip(xort, ans)))
flag = plain.split(b"\x00", 1)[0]
print(flag.decode())
```

公开常量表尾部还包含非 flag 数据，所以应在首个 NUL 处停止，而不是按更长的 `answer` 长度盲目索引。结果为：

```text
miniLCTF{Y0u_s0Lv3d_th3_0bfs?}
```

## 方法总结

这种混淆的明显特征是分发变量与标志寄存器被反复保护。既可模拟分发并重建 CFG，也可在动态写入点抓取最终简化表达式。恢复公式后要根据真实输入长度或终止符截断，避免把相邻常量误当作 flag。
