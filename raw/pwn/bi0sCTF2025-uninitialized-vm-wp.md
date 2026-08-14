# Uninitialized VM

## 题目简述

题目提供一个自定义 VM，按字节码执行。程序逻辑是：先输入长度 `len`，再读入 `len` 字节字节码，通过 `switch(*pc)` 执行指令；遇到未知 opcode 会触发 VM 扩容重建（`expand()`）。

关键约束是：
- `CPY` 只用 8 位长度参数进行 `memcpy`；
- `expand()` 对 `regs` / `memory` 采用全新堆块并重算 `sp/bp`；
- `regs` 里既有 `pc/sp/bp`，也有 `uint64_t regs[8]`，可被字节码构造参与算术和地址移动。

题目是 `no canary + PIE + NX`，并提示 `No canary` + `NX enabled`，典型利用路径是内存布局重定向 + 栈迁移 + ROP。

## 解题过程

### 关键观察

`CPY` 的实现为：

```c
r64_1 = mctx->data_mem + (rctx->regs[idx1] % MAX_STACK);
r64_2 = mctx->data_mem + (rctx->regs[idx2] % MAX_STACK);
r8_1  = *(++pc) % MAX_STACK - 1;
memcpy(r64_1, r64_2, r8_1);
```

长度字段是 `uint8_t`（实际最大到 `255`，`0` 被编码为 `255`），可实现越界复制。`expand()` 里先复制旧 `reg` 再计算新 `sp`：

```c
rctx->bp = &mctx->data_mem[MAX_STACK-1];
rctx->sp = rctx->bp - ((*reg)->bp - (*reg)->sp);
```

这意味着在二次执行（默认分支）后，`sp/bp` 能被算术重定位到非预期区域，配合溢出复制可形成栈/指针伪装。

### 利用链

1. 发送第一段字节码，制造 `expand` 前后状态

脚本通过 `start()` 与 `sla(b'lEn', str(len))` 分批喂入字节码。官方 exploit 在第一段里用 `MOV_R_X / CPY` 等指令写 `0xff`/`0xf7` 这类高位常量，并把长度设置为 `len(payload)+len(payload)`（故意超过一次读取），触发后续分支进入 `expand`。

2. 重定位/泄露阶段

后续阶段反复：
- 用 `POP_R`、`AND`、`OR` 规整指针高位；
- 用 `SUB/ADD` 计算环境指针/基址；
- 用 `CPY` 在 `data_mem` 中搬运 8*10、8*9 等字节块到目标槽位；

short-writeup 中对应点是“use memcpy size overflow to copy unsorted bin pointers onto VM stack”，并“relocate VM stack over environ to leak stack address”。这与脚本中多次通过位运算恢复页基址并反复重建地址关系相吻合。

3. 伪造 ROP 调度数据

最后脚本构造 `rop = ROP(libc)`，在最后字节码段写入 `system`、`/bin/sh`、`pop rdi; ret`、`ret` 地址，并用连续 `PUSH_R / MOV_R_X / SUB / ADD` 组合把它们按固定偏移写入到可返回路径处。

```python
info(f"SYSTEM: {hex(libc.sym['system'])}")
...
sl(b"echo '$$'")
ru(b'$$\n')
sl(b'cat flag.txt')
flag = ru(b'}').decode()
```

4. 触发返回覆盖

脚本后半段最终以 `cat flag.txt` 的方式验证 shell 是否建立，说明利用目标是把 VM 控制流转入用户态 ROP，拿到命令执行通道。

### 验证

官方 exploit 同时支持本地与远程参数（`RE`）。其验证语义固定：触发一轮 shell 后执行 `echo '$$'` 与 `cat flag.txt`，并截断到 `}`。这与官方给的 flag 格式匹配可直接判定链路成立。

## 方法总结

- 核心技巧：`expand()` + `CPY` 的组合提供了“结构可重建 + 长度可越界写”双重原语，先影响 VM 寄存器栈布局，再转为用户态 ROP。
- 识别信号：看到 `unknown opcode -> recreate internal context`、`sp/bp` 重算、以及 `memcpy` 长度可控且不足位宽时，优先评估是否能形成栈/指针映射与地址重定位。
- 复用要点：该题的关键不是单点越界，而是连续两类原语的级联：先污染 VM 自身上下文，再把污染持续化到下一次执行流程。
