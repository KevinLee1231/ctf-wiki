# backdoor

## 题目简述

题目把 QEMU 启动的 Linux 内核作为 TCP 服务暴露，登录凭据为 `ctf:ctf`。启动参数明确启用了 KASLR、KPTI、SMEP 与 SMAP，因此不能依赖固定内核地址，也不能在内核态直接跳到用户页。题目额外提供系统调用号 `1337`：官方利用把一段用户态机器码的地址和长度传给它，说明该调用会在内核上下文执行可控代码，是本题决定性的内核执行 primitive。

官方利用并不调用现成的 `commit_creds(prepare_kernel_cred(0))`，而是从当前任务的 `cred` 结构直接把身份字段清零，再安全返回用户态读取 flag。

## 解题过程

### 建立可靠的内核基址

官方汇编首先执行 `rdmsr` 读取 `IA32_LSTAR`（MSR `0xc0000082`）。该寄存器保存 x86-64 `syscall` 的内核入口地址；用其减去已知的 `entry_SYSCALL_64` 偏移即可得到本次启动的内核基址，绕过 KASLR。

随后按该基址定位 `init_task`，沿 `task_struct.tasks` 双向链表遍历，以进程名 `pwn` 匹配当前利用进程。找到当前 `task_struct` 后，读取其 `cred` 指针，并将 uid、gid、euid、egid 等连续身份字段置零。核心逻辑可概括为：

```nasm
rdmsr                         ; IA32_LSTAR
shl rdx, 32
or  rax, rdx
sub rax, ENTRY_SYSCALL_OFFSET ; rax = kernel base

; 从 init_task 的 tasks 链表找到 comm == "pwn" 的任务
; rbx = current->cred
mov qword [rbx + 0x08], 0
mov qword [rbx + 0x10], 0
mov qword [rbx + 0x18], 0
mov qword [rbx + 0x20], 0
```

### 返回用户态并读取 flag

利用在调用 `1337` 前完成三件事：将进程名设为 `pwn` 以便内核侧定位；以固定地址映射一段用户栈；注册 `SIGSEGV` 处理器。内核 shellcode 修改凭据后，准备用户态的 `RIP/CS/RFLAGS/RSP/SS` 返回帧，按官方汇编中的 CR4 掩码处理访问保护位，再执行 `swapgs; iretq` 回到用户态。官方说明中特意保留信号处理器，用于处理返回过程中与页表切换有关的异常状态。

回到用户态后，只需以已获得的 root 身份打开 `/flag.txt`，读取并写到标准输出。这里的重点是先通过 LSTAR 处理地址随机化，再按正确的任务链和凭据布局提权；直接把固定地址或单纯的用户态 shellcode 当作内核返回地址都会被 KASLR、KPTI 或 SMEP/SMAP 阻断。

### 验证

官方 `exploit.S` 的末尾用户态例程依次执行 `open("/flag.txt")`、`read`、`write`。因此验证条件是：内核代码已将当前进程凭据改为 root，`iretq` 能恢复到用户页，且 flag 内容被打印。本文仅依据仓库中官方汇编与说明整理，未在本地启动 QEMU 或执行 shellcode。

## 方法总结

- 核心技巧：把 LSTAR 泄漏转换为内核基址，再遍历任务链直接修改当前 `cred`。
- 识别信号：内核题提供自定义 syscall、可控内核执行入口，同时 QEMU 参数出现 `kaslr`、`kpti`、`+smep`、`+smap` 时，应优先规划泄漏和受保护的返回链。
- 复用要点：内核提权的关键不只是执行代码；还要确认任务定位、`cred` 布局、用户态返回帧以及 KPTI/SMEP/SMAP 的恢复顺序都与给定内核版本一致。
