# VectorizedMirage-revenge

## 题目简述

附件是一个只校验 16 字节明文块的 Windows 程序，既接受裸 16 字节，也接受 `miniL{<16 bytes>}` 格式。核心 dispatcher 被 CodeVirtualizer 保护，可将 `VIRTUALIZER_START` 到 `VIRTUALIZER_END` 之间视为黑盒；但题目故意保留了一批未内联的普通 handler，包括 VM 栈/状态读写和三个白盒查表边界 `h_wb_init`、`h_wb_mid`、`h_wb_final`。

通过 hook 这些可见 handler，可以完全绕过对 dispatcher 实现的理解，并从运行时输入、输出和立即数中恢复一个 14 轮白盒 AES-256 网络。题目的预期路线是“hook 白盒表 → 拟合等价 AES 语义 → DFA 恢复末轮密钥 → 回推明文”。

## 解题过程

### 从可见 handler 恢复 VM 骨架

虽然 dispatcher 被虚拟化，它仍然要将 opcode 分派给 handler。在 handler 入口和出口记录下列字段就足以还原语义：

```text
handler 类型与立即数
从 VM 栈弹出的值和压回的值
state[16] / scratch[32] 读写索引
白盒表的输入字节和输出字节
```

基础指令可归纳为 `LOAD_STATE`、`STORE_STATE`、`LOAD_SCR`、`STORE_SCR`、`XOR`、`HALT` 和若干噪声指令。一次完整运行的有效调用序列非常规整：

```text
16 次 WB_INIT
13 轮 WB_MID
1 轮 WB_FINAL
最终比较
```

每轮 `WB_MID` 对一列的每个输出字节取 4 项单字节查表结果异或，暂存到 `scratch` 后再整体搬回 `state`。“单字节查表 → 四项 XOR 合并 → 新列”正是白盒 AES 中将 SubBytes 和 MixColumns 融入 T-box 的常见形态。

### 先逆掉最终比较

比较层不按原顺序直接对比 16 字节，而是先置换并异或掩码：

```c
index = (i * 5 + 3) & 15;
mask  = rotl8(0x71 + 0x33 * i, i & 7);
diff |= (output[index] ^ mask) ^ blob[i];
```

对置换和 mask 求逆，得到白盒 AES 的目标输出块：

```text
379c79a0c4f1556975b5418fceab88fa
```

后续问题因而变成：求 16 字节 $P$，使白盒网络的输出等于该块。

### 拟合三类白盒表

运行时可观测到三类表：

```text
kInit[16][256]
kMid[13][4][4][4][256]
kFinal[16][256]
```

其维度与 AES-256 的 16 字节 state、13 个中间轮和 1 个末轮完全对应。

`kInit[i]` 对 0–255 的输入唯一拟合为：

```text
kInit[i][x] = x XOR a0[i]
```

因而 `kInit[i][0]` 就是该字节的初始轮密钥。恢复值为：

```text
05 93 0f 0d af dd dd 5a dc 15 3f c0 4e 22 57 1d
```

`kFinal[out]` 可唯一拟合为：

```text
kFinal[out][x] = Sbox[x XOR a[out]] XOR b[out]
```

`a` 是末轮的字节密钥掩码，`b` 是附加输出编码。这解释了为什么把表直接当成标准 AES 末轮时会多出一个常量。

对每个中间表，都可拟合为：

```text
kMid[r][c][source_row][output_row][x]
  = MC[output_row][source_row] * Sbox[x XOR a_r[source]]
    XOR b_r[c, output_row, source_row]
```

乘法在 $GF(2^8)$ 中进行，拟合得到的 `MC` 精确等于 AES MixColumns 矩阵：

```text
2 3 1 1
1 2 3 1
1 1 2 3
3 1 1 2
```

同一列中的四项常量 `b_r` 会被 XOR 为该输出字节的聚合常量 `d_r`。至此已经将黑盒 VM 还原为可计算的等价 AES-256 网络。

### 用 DFA 恢复末轮密钥

推荐在倒数第二轮的 `WB_MID` 写回 `scratch` 之前，或在最后一轮之前的 `state` 某字节注入故障。倒数第二轮一列上的单字节故障 $e$ 经 MixColumns 传播后，会形成以下四种差分模式之一：

```text
(2e,  e,  e, 3e)
( e, 2e, 3e,  e)
( e, 3e, 2e,  e)
(3e,  e,  e, 2e)
```

先用已拟合的 `b[out]` 去掉末轮输出编码，再对正常/故障密文应用经典 AES DFA 约束，可逐列筛选末轮轮密钥 `K14`。有了 `K14` 后可逆 AES-256 key schedule，也可直接利用已恢复的白盒表逐轮求逆。

### 逐层逆白盒网络

对目标输出 `C` 的每个字节，先按 ShiftRows 映射逆末轮：

```text
source = ((column + row) & 3) * 4 + row
out    = column * 4 + row
before_final[source] = InvSbox(C[out] XOR b[out]) XOR a_final[source]
```

对 13 个中间轮由后向前重复：

```text
W[out]    = state[out] XOR d_r[out]
Z[column] = InvMixColumns(W[column])
prev[src] = InvSbox(Z[src]) XOR a_r[src]
```

最后逆掉 `kInit`：

```text
plain[i] = state[i] XOR a0[i]
```

得到 16 字节：

```text
59 30 75 5f 52 5f 56 4d 5f 4d 61 73 74 33 72 21
```

即：

```text
Y0u_R_VM_Mast3r!
```

完整 flag 为：

```text
miniL{Y0u_R_VM_Mast3r!}
```

对发布版程序的黑盒验证结果：

```text
miniL{Y0u_R_VM_Mast3r!} -> accepted
Y0u_R_VM_Mast3r!         -> accepted
miniL{AAAAAAAAAAAAAAAA}  -> rejected
```

原始题解提到的强网杯 `ez_vm` 只是 hook 虚拟化边界思路的参考；本文已将本题需要的 handler 边界、白盒表形状、AES 拟合、DFA 故障模式和逐轮求逆全部写入正文，不依赖外链才能复现主线。

## 方法总结

- 核心技巧：不进入 CodeVirtualizer dispatcher，改在稳定的未虚拟化 handler 边界做动态语义恢复，再将查表网络拟合为白盒 AES-256 并使用 DFA/逆网络求明文。
- 识别信号：一组 `noinline` 小 handler 反复读写 `state/scratch`，三块 256 项字节表呈现 `16 + 13×4×4×4 + 16` 的形状，并在中间轮以四项 XOR 形成列，应优先怀疑白盒 AES。
- 复用要点：先逆最终比较取得稳定目标块，再拟合表的代数形式；白盒附加编码不先剔除会破坏 DFA 约束，因此必须分离 `a` 密钥掩码与 `b/d` 输出编码常量。
