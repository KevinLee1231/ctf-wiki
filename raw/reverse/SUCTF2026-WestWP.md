# SUCTF2026-West

## 题目简述

题目给出一个 Windows x64 程序，要求依次提交 81 个十进制数；每个数必须恰好为 16 位。程序不会按层号顺序验证，而是通过一张 81 项置换表选择本轮的验证函数。每层都带有独立常量和一段经 SplitMix64 解码的 NOR VM 程序，验证成功后还会继续更新全局状态并逐轮解密 40 字节 flag。

这是一道典型的 auto-re：单层逻辑可逆，但 81 层之间存在状态依赖。稳定的解法是先抽象公共模板，再逐轮求出输入，并用真实或等价实现推进状态；不能把 81 个函数彼此独立地求解。

## 解题过程

### 1. 恢复输入与调度关系

程序支持两种输入方式：命令行参数中的 81 项 CSV，或交互式逐项输入。两种方式最终都要求每项满足：

```text
1000000000000000 <= x <= 9999999999999999
```

主验证循环可还原为：

```c
for (size_t round = 0; round < 81; round++) {
    uint8_t layer = call_schedule[round];
    ctx.round_index = round;

    if (anti_debug_guard(&ctx, round, layer, input[round], PRE_TAG, false))
        return false;
    if (!layer_dispatch[layer](&ctx, input[round]))
        return false;
    if (anti_debug_guard(&ctx, round, layer,
                         ctx.state ^ input[round], POST_TAG, false))
        return false;
}
```

实际调度顺序从 0 开始为：

```text
0,75,31,46,52,71,27,57,1,25,40,67,42,10,64,16,66,49,3,45,4,2,72,
43,18,76,48,21,77,60,65,63,70,47,39,79,23,59,50,5,61,7,78,62,51,
29,44,74,15,24,14,32,80,55,56,68,35,73,37,53,69,6,36,28,13,38,17,
26,33,8,22,54,11,9,34,30,19,41,12,58,20
```

官方源码中的运行时状态为：

```c
typedef struct {
    uint64_t state;
    size_t round_index;
    uint64_t anti_acc;
    uint32_t tamper_score;
    uint8_t flag_data[40];
} RuntimeCtx;
```

因此反编译结果中偏移 `0x00`、`0x08`、`0x10`、`0x18` 和 `0x1c` 分别对应上述字段。初始状态为 `0x669e1e61279d826e`，`flag_data` 初始存放的是 40 字节密文。

### 2. 抽象每一层的公共模板

81 个分发函数虽然变量名和附加干扰代码不同，决定验证结果的主干完全一致：

```c
mixed = transform_round_input(raw_input, ctx->state, round, layer, layer_id);
pre   = preprocess_input(mixed, ctx->state, round, layer, layer_id);
vm    = run_nor_vm(pre, mixed, ctx->state, round, layer);
check = build_check(pre, vm, ctx->state, round, layer);

if (check != layer->target)
    return false;

round_key = derive_round_key(pre, vm, ctx->state, round, layer);
decrypt_flag_round(ctx->flag_data, round_key, round);
ctx->state = update_state(pre, vm, ctx->state, mixed, round, layer, layer_id);
```

对应到比赛二进制中的主要 helper：

| helper | 作用 |
|---|---|
| `sub_140012480` | 把原始输入异或层种子后，执行 32 位双分支 Feistel/ARX 混合 |
| `sub_140012630` | 对 64 位值反复执行异或、循环左移和加法预处理 |
| `sub_140012780` | 用 SplitMix64 解码本层指令，再执行以 `~(a | b)` 为核心的 NOR VM |
| `sub_140012940` | 对预处理结果执行 3 或 4 轮可逆 ARX，生成待比较值 |
| `sub_140012B90` | 结合 VM 输出、预处理值和当前状态派生轮密钥 |
| `sub_140012C00` | 用轮密钥原地解密 40 字节 flag |
| `sub_140012CA0` | 更新下一轮使用的 64 位状态 |

第一段是 Feistel 结构。将输入与 `enc_seed` 异或后拆成两个 32 位半部，每轮只需根据右半部计算混合量：

```python
left, right = high32(x ^ enc_seed), low32(x ^ enc_seed)
for step in range(input_rounds + 6):
    k = input_consts[step & 3] ^ base ^ ((step + 1) * DELTA)
    mix_a = rol32(right ^ low32(k), shift) + (high32(k) ^ right)
    mix_b = low32(k) + ror32(right, ((shift + 5) % 31) + 1)
    left, right = right, left ^ mix_a ^ mix_b
```

第二段是标准可逆 ARX 链：

```python
value ^= xor_const
value = rol64(value, shift)
value += add_const
```

第三段 `build_check` 仍是 3 或 4 轮同类 ARX。源码中该函数明确不使用传入的 `vm_value` 和 `state`；当前状态只会通过前两段间接影响 `pre`。这意味着“求当前输入”的约束只需覆盖前三个可逆 helper：

```text
build_check(preprocess_input(transform_round_input(x))) == target[layer]
```

NOR VM 仍然不能省略。它虽然不参与本轮的目标比较，却参与轮密钥和下一轮状态的计算；若只求出 81 个局部解而不执行 VM，第二轮开始使用的状态就会错误，最终也无法解出 flag。

### 3. 逐轮求解并推进状态

官方解法使用 Z3。对每轮建立一个 64 位变量 `x`，限制其为 16 位十进制数，把 Feistel、预处理 ARX 和检查 ARX 按位精确翻译成 BitVec 表达式：

```python
inputs = []
state = INITIAL_STATE
flag = bytearray(ENCRYPTED_FLAG)

for round_idx in range(81):
    layer_id = CALL_SCHEDULE[round_idx]
    spec = LAYERS[layer_id]

    x = BitVec(f"x_{round_idx}", 64)
    s = Solver()
    s.add(UGE(x, BitVecVal(10**15, 64)))
    s.add(ULE(x, BitVecVal(10**16 - 1, 64)))

    mixed = transform_round_input_sym(x, state, round_idx, spec, layer_id)
    pre = preprocess_input_sym(mixed, state, round_idx, spec, layer_id)
    check = build_check_sym(pre, round_idx, spec)
    s.add(check == BitVecVal(spec.target, 64))
    assert s.check() == sat

    value = s.model().eval(x).as_long()
    inputs.append(value)

    # 求解后必须用整数实现完整执行本轮。
    mixed_c = transform_round_input(value, state, round_idx, spec, layer_id)
    pre_c = preprocess_input(mixed_c, state, round_idx, spec, layer_id)
    vm_c = run_nor_vm(pre_c, mixed_c, state, round_idx, spec)
    key = derive_round_key(pre_c, vm_c, state, round_idx, spec)
    decrypt_flag_round(flag, key, round_idx)
    state = update_state(pre_c, vm_c, state, mixed_c,
                         round_idx, spec, layer_id)
```

这里的旋转、加法和乘法都必须在对应位宽上截断；尤其不能用 Python 的无限精度整数直接代替 32/64 位溢出。每求出一轮后，先用具体值重新计算并断言 `check == target`，再推进状态，可以尽早定位常量提取、轮号或层号写反的问题。

参赛队总 WP 还给出了另一条等价路线：直接逆转 `sub_140012940`、`sub_140012630` 和 Feistel 结构，得到本轮输入；随后用 Unicorn 映射原 PE，按 Win64 ABI 调用分发函数，让二进制负责 VM、flag 和状态更新。这条路线省去对完整 VM 的手写复现，但要正确映射 PE、栈和 shadow space，并且只能桩掉与环境相关的反调试 helper，不能跳过核心分发函数。

逐轮求解并完成 81 次 flag 解密后得到：

```text
SUCTF{y0u_h4v3_0v3rc0m3_81_d1ff1cu1t135}
```

## 方法总结

- 面对大量外观不同的分发函数，先找共同 helper、层常量表和调度表，不要逐函数手工抄写。
- 本题的输入约束由 Feistel 和两段可逆 ARX 决定；NOR VM 不参与目标比较，但决定轮密钥和后续状态，仍需具体执行。
- 81 轮必须按 `call_schedule` 顺序求解。轮号、层号和输入下标是三个不同概念，混用会从第一轮或第二轮开始产生错误。
- 可以选择“Z3 求当前输入 + Python 具体推进状态”，也可以“解析逆变换 + Unicorn 推进状态”；两者的共同验收标准都是每轮目标比较成立，且最终 40 字节明文为完整 flag。
