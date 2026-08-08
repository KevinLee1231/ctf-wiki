# GQuuuuuupX

## 题目简述

附件是一个静态链接、无 section header 的 x86-64 Linux ELF，外层由 UPX 打包。它要求输入 `miniL{...}`，花括号中的 body 长度固定为 103，字符集为 `[A-Z0-9_]`。

这不是普通的“解 UPX 后逆向”：`upx -d` 生成的程序使用默认密钥 `0x42`，只接受诱饵 flag；原始 packed 程序的 stub 在跳转到 OEP 前把同一个全局字节改成 `0x37`，从而选择另一套 VM/ARX 参数和真实校验材料。决定性主障碍是识别这个解包前后的运行时状态差异，再恢复逐字节可逆的校验链。

## 解题过程

### 先解出诱饵路径

`upx -d` 可正常得到 stripped ELF。在解包程序中可找到如下 profile 选择逻辑：

```c
g_stub_state.key = 0x42;
profile = ((((unsigned)g_stub_state.key >> 1) ^ g_stub_state.key) ^ 1) & 1;
```

因而解包程序使用 `key=0x42, profile=0`。校验数据不是明文字符串，而是编码在 `g_material_blob`、`g_round_program_enc` 和 `g_opcode_map_enc` 中的 masked anchors、轮常量、VM 字节码和 opcode 映射。每一位的恢复流程可概括为：

```text
raw       = masked_anchor[i] ^ anchor_mask(profile, key, i, rolling)
step      = derive_step(profile, key, i, state, scratch, rolling, raw)
target    = step 的低 8 位
body[i]   = invert_transform_byte(target, state, step, i)
rolling   = update_rolling(..., raw, body[i], target)
state     = mix_body_byte(..., body[i], step)
```

`transform_byte()` 只由模 $2^8$ 加减、异或和 8 位循环移位构成，可以严格倒序。用 `profile=0, key=0x42` 解出的诱饵为：

```text
miniL{ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL_1FAEFB6177B4672DEE07F9D3AFC62588CCD2631EDCF22E8CCC1FB35B501C9C86}
```

它能通过 `upx -d` 后的程序，却会被原始 packed 程序拒绝。这个对照是继续调查 UPX stub 的关键证据，不能把诱饵当成最终答案。

### 监视 stub 对密钥的改写

解包 ELF 是 non-PIE，全局状态字节地址为 `0x407fa0`。对原始附件下硬件 watchpoint：

```gdb
watch *(unsigned char*)0x407fa0
run miniL{A}
continue
```

第一次改写是 loader 把默认值设为 `0x42`；第二次会在 stub 尾部观察到：

```text
Old value = 0x42
New value = 0x37
0x403a33: movb $0x37,-0x60(%r13)
0x403a3a: jmp *%rax
```

题目的 `patch_stub.py` 也印证了这一点：它在 UPX stub 内嵌入 `0f0541c645a0375a58ffe090` 特征，完成字节 `0x37` 写入后跳转 OEP。代入 profile 公式得到：

```text
key = 0x37
profile = 1
```

这个变化会同时影响 VM 程序解码、opcode map、material slot 顺序、scratch 大小、anchor mask 和 rolling state，所以不是简单替换一个字符串常量。

### 恢复真实 body

使用与上述校验器同构的恢复器，但切换到 `profile=1, key=0x37`。核心循环必须在恢复每个字节后立即更新 `rolling/state/scratch`，否则第二字节起就会与原程序分歧：

```python
def recover_body(profile, key, masked_anchors):
    state, scratch, program, opcode_map, rolling = init_profile(profile, key)
    output = []
    for i, masked_anchor in enumerate(masked_anchors):
        raw = masked_anchor ^ anchor_mask(profile, key, i, rolling)
        step, target = derive_step(profile, key, i, state, scratch, rolling, raw)
        byte_value = invert_byte(target, state, step, i)
        output.append(byte_value)
        rolling = update_rolling(
            profile, key, i, state, scratch, rolling, raw, byte_value, target
        )
        mix_body_byte(
            state, scratch, profile, i, byte_value, step,
            ROUND_CONST, program, opcode_map
        )
    return bytes(output).decode()
```

恢复并提交以下结果：

```text
miniL{HELLO_FROM_THE_OTHER_SIDE_IMUSTVE_CALLED_THOUSAND_TIMES_TO_TELL_YOU_IM_SORRY_FOR_EVERYTHING_THAT_I_DONE}
```

官方验收逻辑分别运行 plain、重新解包和 packed 三个文件：plain/unpacked 必须接受 decoy 并拒绝 real，packed 则必须接受 real 并拒绝 decoy。

若不想完整重建 ARX/VM，也可在逐字节比较点下断点，对 `[A-Z0-9_]` 逐位枚举。由于校验器是流式状态机，每轮用已确定前缀启动新进程，在第 $i$ 次比较时观察是否命中，同样可以恢复 103 字节 body。

## 方法总结

- 核心技巧：对比 packed 与 unpacked 程序的行为，通过 watchpoint 捕获 UPX stub 在 OEP 前的状态注入，再按真实 profile 逆转逐字节 ARX 校验链。
- 识别信号：`upx -d` 后的正确输入被原附件拒绝，说明壳不仅负责解压，还可能修改原程序数据或密钥。
- 复用要点：先用黑盒对照证明解包产物是诱饵，再监视关键全局状态；对流式校验器，不论选择算法逆向还是逐位枚举，都必须精确重放前缀状态。
