# d3bpf

## 题目简述

此题是一个 Linux kernel eBPF 利用入门题。主要参考 CVE-2021-3490 一类 eBPF verifier 边界追踪错误：Ubuntu CVE 条目 https://ubuntu.com/security/CVE-2021-3490 将其概括为 eBPF ALU32 bitwise operation 的 bounds tracking 没有正确更新 32-bit 范围，攻击者可以构造通过 verifier 的程序，在运行时造成越界读写并进一步提权。本题不是原样复现 CVE-2021-3490，而是在 `BPF_RSH` 右移处理上人为加入类似“verifier 认定值为 0、运行时值仍为 1”的不一致。

## 解题过程

### 1.Analysis

当 CONFIG_BPF_JIT_ALWAYS_ON  生效时（本题的 kernel 就是这样的），ebpf 字节码在载入内核后，会首先通过一个 verifier 的检验。保证不存在危险后，会被 jit 编译为机器指令。然后触发时，就会执行jit 后的代码。

因此，如果 verifier 判断出错，就可能通过 ebpf 注入非法代码，实现权限提升。

附件中提供了 diff 文件，可以发现添加了一个漏洞

```
...
diff --git a/kernel/bpf/verifier.c b/kernel/bpf/verifier.c
index 37581919e..8e98d4af5 100644
--- a/kernel/bpf/verifier.c
+++ b/kernel/bpf/verifier.c
@@ -6455,11 +6455,11 @@ static int adjust_scalar_min_max_vals(struct
bpf_verifier_env *env,
                        scalar_min_max_lsh(dst_reg, &src_reg);
                break;
        case BPF_RSH:
-               if (umax_val >= insn_bitness) {
-                       /* Shifts greater than 31 or 63 are undefined.
-                        * This includes shifts by a negative number.
-                        */
-                       mark_reg_unknown(env, regs, insn->dst_reg);
+               if (umin_val >= insn_bitness) {
+                       if (alu32)
+                               __mark_reg32_known(dst_reg, 0);
+                       else
+                               __mark_reg_known_zero(dst_reg);
                        break;
                }
                if (alu32)
...
```

在 X86_64 下，对于 64 位寄存器进行右移操作时，如果操作数大于 63，那么大于 63 的部分会被忽略（也就是只有操作数的低 6 位是有效的）。那么如果我们进行这样一个操作 BPF_REG_0 >> 64 （且通过了 verifier 的检测），在 ebpf 代码通过 jit 编译后，生成的汇编代码就可能是这样的: shr rax, 64 ，代码执行后，BPF_REG_0  应当仍然保持为 1。

不过这是架构相关的，不同架构可能会有不一样的表现，所以我们可以看到，patch 前的 verifier 在处理该操作时，会把寄存器的范围设置为 unknown。然而 patch 后的 verifier 则会把寄存器直接置为 0，如果我们提前把寄存器置为 1，对它执行右移 64 位的操作，之后就会获得一个运行时值为 1，但是verifier 确信为 0 的寄存器。

### 2. exploit

在我们获得了这样的寄存器后，就可以绕过 verifier 对指针运算的范围检测。很容易实现，假设

EXP_REG  是一个运行时值为 1，而 verifier 认定为 0 的寄存器，只要将 EXP_REG  乘以任意值，与一个指针相加，verifier 会认为将会执行的运算为 ptr + 0 ，实际上则会是 ptr + arbitrary_val 。

不过，在 ebpf 字节码通过 verifier 的检测后，还会通过 fixup_bpf_calls  给字节码添加一些 patch （这个 patch 指，对传入的 ebpf 字节码，在某些存在危险的操作前添加的一些 bpf 指令。添加的字节码将在 jit 编译后一起被加入代码中，作为一种运行期的检测）才会 jit 生成代码，在这里，对于指针（ptr）和标量（scalar）的 BPF_ADD 或 BPF_SUB 运算，会添加这样的 patch

```
...
            if (isneg)
                *patch++ = BPF_ALU64_IMM(BPF_MUL, off_reg, -1);
            *patch++ = BPF_MOV32_IMM(BPF_REG_AX, aux->alu_limit - 1);
            *patch++ = BPF_ALU64_REG(BPF_SUB, BPF_REG_AX, off_reg);
            *patch++ = BPF_ALU64_REG(BPF_OR, BPF_REG_AX, off_reg);
            *patch++ = BPF_ALU64_IMM(BPF_NEG, BPF_REG_AX, 0);
            *patch++ = BPF_ALU64_IMM(BPF_ARSH, BPF_REG_AX, 63);
            if (issrc) {
                *patch++ = BPF_ALU64_REG(BPF_AND, BPF_REG_AX,
                             off_reg);
...
```

这里的 off_reg 指的是将要和 ptr 相加的 scalar。alu_limit 是该指针能够接受的运算的最大值（verifier对 ptr 的值做跟踪，从而计算出保证 ALU 运算后不会溢出的最大值）。patch 后的代码在执行时，如果off_reg 的值大于 alu_limit，或者两者的符号相反，那么 off_reg 就会被置 0，可以认为指针运算就不会发生。

但是此时我们有一个运行时值为 1，而 verifier 认定为 0 的寄存器，所以其实很容易绕过这个 patch。

```
    BPF_MOV64_REG(BPF_REG_0, EXP_REG),
    BPF_ALU64_IMM(BPF_ADD, OOB_REG, 0x1000),
    BPF_ALU64_IMM(BPF_MUL, BPF_REG_0, 0x1000 - 1),
    BPF_ALU64_REG(BPF_SUB, OOB_REG, BPF_REG_0),
    BPF_ALU64_REG(BPF_SUB, OOB_REG, EXP_REG),
```

这里 OOB_REG 是一个指向一个 oob_map 头部的指针，EXP_REG 是运行时值为 1，而 verifier 认定为0 的寄存器，先给 oob_map 加 0x1000，然后通过 EXP_REG 将指针减回 oob_map 头部，verifier 仍然会认为 OOB_MAP 指向的是 &oob_map + 0x1000 ，所以之后对 OOB_MAP 做减法时，patch 的alu_limit 仍然会是 0x1000，就可以实现向低地址溢出。

可以实现 oob 后，最直接的就是我们可以实现 leak，在 oob_map 前面是一个 bpf_map 的元数据，其中存储了一个虚表的地址，该虚表处于内核的 .text 段，load 出来即可实现 leak。

```
        BPF_ALU64_IMM(BPF_MUL, EXP_REG, OFFSET_FROM_DATA_TO_PRIVATE_DATA_TOP),
        BPF_ALU64_REG(BPF_SUB, OOB_REG, EXP_REG),
        BPF_LDX_MEM(BPF_DW, BPF_REG_0, OOB_REG, 0),
        BPF_STX_MEM(BPF_DW, STORE_REG, BPF_REG_0, 8),
        BPF_EXIT_INSN()
```

任意地址读可以通过 obj_get_info_by_fd 函数实现。该函数会返回 bpf->id 的值。通过溢出修改 btf 指针即可任意地址读。

```
//kernel/bpf/syscall.c
    if (map->btf) {
        info.btf_id = btf_obj_id(map->btf);
        info.btf_key_type_id = map->btf_key_type_id;
        info.btf_value_type_id = map->btf_value_type_id;
    }
```

通过劫持 oob_map 元数据中的虚表，可以实现任意函数调用。为了劫持虚表，我们需要向内核中写入一些数据，并且需要知道该数据的地址，笔者通过内核的基数树实现从 init_pid_ns  开始搜索，搜索到本进程的 task_struct，然后获取 fd_table，然后找出 bpf_map 的地址，读出 bpf_map-

>private_data  的值，由此获得了一个 map 的地址，然后写入 work_for_cpu_fn  劫持

map_get_next_key  指针，调用此函数，即可实现 commit_cred(&init_cred)  实现提权。

搜索基数树的代码比较长，这里就不放了，完整的 exp 在我的 GitHub 仓库中。

### 3. more...

正如文章开头所写，本题出题前主要参考了对 CVE-2021-3490 的利用，事实上笔者也是 kernel 利用的初学者，在学习了该利用后才出的这道题。由于新版本的 kernel 增加了对 ALU 运算的 mitigation，所以此利用方法其实已经失效，为了出题选用了一版本较旧的内核，却忘了 patch 掉 CVE-2021-3490，有些师傅使用该 CVE 的公开 exp 改改偏移就打通了，给各位大师傅们带来了十分不好的做题体验，在这里献上笔者最诚挚的歉意。

虽然新版本添加了 mitigation，但是对于类似的漏洞，仍然是可利用的，请看 d3bpf-v2。

## 方法总结

- 核心技巧：制造一个运行时为 1、verifier 认为为 0 的寄存器，用它绕过 verifier 和 ALU sanitizer 对指针偏移的限制。
- 利用链：先通过 map 元数据越界读泄露内核基址，再借 `obj_get_info_by_fd` 和篡改 `btf` 指针实现任意地址读，最后劫持 bpf map 虚表函数指针调用 `commit_creds(&init_cred)`。
- 外部背景：CVE-2021-3490 的本质是 eBPF verifier 对寄存器范围的静态认知与 JIT 后真实执行结果不一致；本题 patch 虽然落点不同，但利用思想相同。
- 复用要点：eBPF 题要分别检查 verifier 静态状态、JIT 机器语义和 sanitizer runtime patch，三者有任何一处认知不一致都可能变成 OOB 原语。
