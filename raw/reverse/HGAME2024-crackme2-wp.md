# crackme2

## 题目简述

程序表面上像是一个 Base64 校验器，真正的验证逻辑却被藏在 Windows SEH 异常处理流程中，并经过 SMC 自修改解密。隐藏代码还会调用 `NtQueryInformationProcess` 检查调试端口。恢复这段代码后，可看到 32 个输入字节满足一组线性方程；将方程交给 Z3 即可得到 flag。

## 解题过程

### 从 SEH 找到隐藏控制流

程序故意向空指针写入数据以触发访问异常：

```c
__try {
    *((char *)0) = 1;
}
__except (EXCEPTION_EXECUTE_HANDLER) {
    /* 真正逻辑位于异常处理路径 */
}
```

若只沿正常控制流静态分析，会停留在 Base64 诱饵。应检查 SEH handler，跟入 `__except` 中的代码。异常处理器查询当前进程的 `ProcessDebugPort`，随后把一段内存改为可读、可写、可执行，使用 `data` 数组逐字节 XOR 解密 `encrypt` 区域，再恢复原页面权限：

```c
DWORD old_protect;
__int64 debug_port;
DWORD returned;

NtQueryInformationProcess(
    GetCurrentProcess(),
    ProcessDebugPort,
    &debug_port,
    8,
    &returned
);

if (debug_port != -1) {
    VirtualProtect(encrypt, 4096 * 6, PAGE_EXECUTE_READWRITE, &old_protect);
    for (int i = 0; i < sizeof(data); i++) {
        ((char *)encrypt)[i] ^= data[i];
    }
    VirtualProtect(encrypt, 4096 * 6, old_protect, &old_protect);
}
```

恢复方式有三种：修补反调试分支后运行到解密完成处并 dump；在调试器中手工执行 XOR；或静态写脚本对 `encrypt` 与 `data` 做异或，再把结果映射回原地址。完成后重新创建函数，即可看到 32 字节 flag 的方程校验。

### 用整数变量解方程

隐藏函数中的每条判断都是各输入字符的整数线性组合。官方题解展示的第一条约束为：

```python
25*flag[0] + 240*flag[1] + 103*flag[2] + 194*flag[3] + \
201*flag[4] + 211*flag[5] + 62*flag[6] + 9*flag[7] + \
96*flag[8] + flag[9] + 212*flag[10] + 207*flag[11] + \
64*flag[12] + 52*flag[13] + 115*flag[14] + 142*flag[15] + \
64*flag[16] + 114*flag[17] + 195*flag[18] + 14*flag[19] + \
192*flag[20] + 45*flag[21] + 96*flag[22] + 36*flag[23] + \
5*flag[24] + 27*flag[25] + 52*flag[26] + 70*flag[27] + \
24*flag[28] + 255*flag[29] + 20*flag[30] + 10*flag[31] == 296473
```

实际求解时，把恢复函数中的全部等式逐条加入 Z3。这里使用 `Int` 比 `BitVec(8)` 更合适：反编译出的运算是提升后的整数线性运算，不需要模拟 8 位溢出；同时加上可打印字符范围和已知 flag 格式可减少歧义。

```python
import z3

solver = z3.Solver()
flag = [z3.Int(f"flag[{i}]") for i in range(32)]

for value in flag:
    solver.add(value >= 0x20, value <= 0x7E)

prefix = b"hgame{"
for index, value in enumerate(prefix):
    solver.add(flag[index] == value)
solver.add(flag[-1] == ord("}"))

# 从恢复后的验证函数逐条录入其余 32 元一次方程。
solver.add(
    25*flag[0] + 240*flag[1] + 103*flag[2] + 194*flag[3]
    + 201*flag[4] + 211*flag[5] + 62*flag[6] + 9*flag[7]
    + 96*flag[8] + flag[9] + 212*flag[10] + 207*flag[11]
    + 64*flag[12] + 52*flag[13] + 115*flag[14] + 142*flag[15]
    + 64*flag[16] + 114*flag[17] + 195*flag[18] + 14*flag[19]
    + 192*flag[20] + 45*flag[21] + 96*flag[22] + 36*flag[23]
    + 5*flag[24] + 27*flag[25] + 52*flag[26] + 70*flag[27]
    + 24*flag[28] + 255*flag[29] + 20*flag[30] + 10*flag[31]
    == 296473
)

# ...继续加入隐藏函数中的其余等式...

assert solver.check() == z3.sat
model = solver.model()
print("".join(chr(model[value].as_long()) for value in flag))
```

官方 PDF 没有逐页列出全部方程，但保留了完整模型结果。按索引重排其 32 个 ASCII 值可得到：

```python
model_values = {
    10: 52, 31: 125, 29: 110, 20: 103, 0: 104, 13: 95,
    12: 100, 2: 97, 25: 52, 28: 79, 11: 110, 8: 67,
    16: 108, 18: 49, 27: 49, 6: 83, 3: 109, 7: 77,
    17: 118, 24: 117, 1: 103, 19: 110, 22: 101, 9: 95,
    15: 48, 5: 123, 21: 95, 30: 115, 4: 101, 23: 113,
    14: 115, 26: 116,
}

flag = "".join(chr(model_values[i]) for i in range(32))
print(flag)
```

输出为：

```text
hgame{SMC_4nd_s0lv1ng_equ4t1Ons}
```

官方特别提醒，若用 32 个 `BitVec` 变量，求解可能耗时很久；把变量建成 `Int` 后，同一组线性方程通常可在很短时间内得到结果。

## 方法总结

- Windows 程序主动制造异常时，不应把异常路径视为错误处理；SEH handler 很可能承载真正控制流。
- `VirtualProtect(..., PAGE_EXECUTE_READWRITE, ...)` 后紧跟逐字节变换，是定位 SMC 解密循环的重要信号。
- 动态 dump、手工 XOR 和静态解密都可以恢复 SMC；关键是保持原加载地址，方便反汇编器正确解析跳转与引用。
- 纯整数线性方程优先使用 Z3 `Int`。只有确实依赖固定位宽回绕、移位或位运算时，才使用 `BitVec`。
- 模型按变量索引输出，并不保证字典顺序就是字符串顺序；必须用 `flag[0]` 到 `flag[31]` 重排后再转 ASCII。
