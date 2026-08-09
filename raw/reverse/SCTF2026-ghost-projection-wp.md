# ghost_projection

## 题目简述

外层 `ghost_abyss_hardened` 先解密并校验一个中间 block，再解密真正的 ELF64 stage2，手动映射其 `PT_LOAD` 段、写入 loader cookie、设置权限并跳到入口。stage2 具有大量反调试、状态镜像和投影 gate，但隐藏 flag 字节最终仍会在 `route_projection_residue_byte()` 返回后、与用户输入异或前短暂出现在 `AL`。

官方解法不逆完整状态机，而是恢复可独立运行的 stage2，修补 gate 让四个 epoch 的 oracle 依次打开，在干净泄漏点注入 logger，按隐藏索引合并四次运行记录。

## 解题过程

### 1. 解出并修复 stage2 ELF

外层在文件偏移 `0x13520` 保存加密 stage2，使用带 xorshift/轮转的字节流和置换索引解密。恢复结果应以 `\x7fELF` 开头，并含合法 ELF64 program header。

原 loader 在映射后向虚拟地址 `0x42f000` 写入：

```text
0x6a09e667f3bcc909
```

直接把内存映像 dump 成文件会遗漏位于 `p_memsz`、但超出原 `p_filesz` 的 cookie。`fix_outer_payload.py` 将对应 `PT_LOAD` 扩展到该地址，补入 cookie 并同步更新 `p_filesz`，生成可直接执行的 `stage2_payload_loaderfixed.elf`。

### 2. 打开 route projection gate

关键 gate 要求：

```text
bias == 1
phase == 0x73
lane == 2
epoch <= 3
((idx XOR epoch) & 3) == 0
runtime/orbit byte 与 epoch 目标一致
```

其中 bias、phase、lane 和 runtime latch 由多组 `enc XOR key` 字段计算。官方脚本修补六个编码字段，使初始化后的 latch 与 gate 条件同时成立；再把 `epoch_gate.epoch_enc` 分别改成 0、1、2、3 对应值。每次运行只泄漏索引满足 $idx\bmod4=epoch$ 的字节，因此四轮恰好覆盖全部 43 个位置。

### 3. 在输入混入前记录原始返回值

最合适的 hook 位于 `0x4141dd`：

```asm
call route_projection_residue_byte
xor  al, byte ptr [rbp+0x20]
```

在第二条指令执行前，`AL` 是尚未混入 staged input 的隐藏字节，`ctx+0x16` 是其 hidden index。把原 6 字节改成跳到 RX code cave `0x4286c4` 的指令，logger 向 stderr 写固定 0x28 字节记录，至少保存：

```text
magic = OCAFLOG0
raw return
logical position
phase
hidden index
staged input byte
```

记录后恢复寄存器，补执行原来的两个 XOR，再跳回 `0x4141e3`。同时扩展 RX `PT_LOAD` 到 `0x28000`，确保 code cave 真正位于文件映像和可执行段内。

### 4. 合并四个 epoch

运行 `recover_flag_patch_oracle.py` 生成并执行四个 ELF。解析 stderr 中的记录，对每条记录取：

```text
flag[hidden_idx] = raw_return & 0xff
```

四轮分别得到 11、11、11、10 个字节，合并后没有缺口：

```text
sctf{fabsfoagf3432y9adl!fesfsffhoyh345gdhh}
```

## 方法总结

本题的关键不是把巨大 `AbyssState` 每个字段都命名完，而是追踪“隐藏数据何时第一次以未掩码字节出现”。外层恢复必须忠实补回 loader 的运行时 cookie；内层则用 gate 条件控制可观测 epoch，并把 hook 放在输入 XOR 之前。脚本对解密 ELF、修复 ELF 和四个 patched ELF 都计算哈希，且 raw stage2 与参考文件一致，这些是比肉眼 dump 更可靠的复现证据。
