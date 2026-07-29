# pewpew

## 题目简述

题目运行在 Windows Server 2025 上，附件提供 `pewpew.exe` 及匹配的 `ntdll.dll`、`kernel32.dll` 和 `KernelBase.dll`。程序模拟飞船控制台，核心对象有两种：

- Scout：大小为 `0x50` 的原始字节缓冲区，可查看、重载和释放；
- Pilot：同样大小为 `0x50`，首字段为 vtable 指针，后续保存对象自身地址等状态。

`launch pilot` 会释放 Pilot 的堆块，却保留全局 Pilot 指针。之后的 identify/scan/tune circuit 仍通过这个悬空指针调用虚函数，形成 UAF。

目标不是简单执行 `cmd.exe`。远端启动器每次把程序解压到随机临时目录，`flag.txt` 与程序位于同一目录；新进程也不会自然继承当前网络输出通道。因此最终利用需要在同一连接中动态取得当前目录句柄、相对打开 `flag.txt`、读取内容，再通过程序自己的 socket 输出函数返回。

## 解题过程

### 通过 LFH 复用 Pilot

Windows Low-Fragmentation Heap 对同尺寸块的复用带有随机性。公开解法先创建约 400 个空 Scout 做堆整理，然后：

1. 创建一个 Pilot；
2. 选择 `launch pilot`，释放其 `0x50` 字节块；
3. 立即创建一个空 Scout；
4. 查看这个 Scout 的内容。

若新 Scout 复用了刚释放的 Pilot，未清零的缓冲区会残留：

```text
offset 0x00 -> Pilot vtable 指针
对象字段     -> Pilot 自身的堆地址
```

vtable 位于 `pewpew.exe`，因此第一项泄漏可恢复 EXE 基址；对象自指针则给出攻击块的堆地址。若 Scout 内容不符合 Pilot 结构，就关闭本次连接重新尝试。堆喷后命中率约为三分之一，符合题目“少于 10 次、不可暴力”的可靠性要求。

### 把可控对象变成读写原语

程序内置的几个 firmware/circuit 辅助函数在正常语义下只是菜单功能，但作为伪 vtable 目标时可组成：

- tap-scan：从对象中的游标地址读取并输出数据，构成任意读；
- readline：从当前 socket 向对象指定区域继续读入攻击者字节，构成受控写；
- WriteFile wrapper：把给定缓冲区写到标准输出，而该服务的标准输出就是当前 socket。

先把复用后的 Scout 重载成伪 Pilot，使 circuit 调用落到 tap-scan。读取 EXE 导入表即可取得 `ntdll.dll`、`kernel32.dll` 和 `KernelBase.dll` 的实际地址。附件提供匹配 DLL，因而可以离线计算所需导出函数和 ROP gadget 的 RVA，再加泄漏基址得到运行时地址。

### 用 `NtContinue` 恢复完整寄存器

通过 vtable 间接调用时，Windows x64 的 `rcx`、`rdx`、`r8`、`r9` 并不都可自由设置。解决方法是调用：

```text
NtContinue(fake_context, FALSE)
```

`NtContinue` 按 `CONTEXT` 结构恢复通用寄存器、`rip` 和 `rsp`。利用 readline 原语在受控堆区域写入：

```text
fake CONTEXT
ROP stack
UNICODE_STRING / OBJECT_ATTRIBUTES
文件名与 IO_STATUS_BLOCK
读文件缓冲区
```

然后把伪 vtable 对应槽位指向 `NtContinue`。触发 circuit 后，线程直接切换到伪造寄存器和栈。

写入载荷时有两个工程约束：

- 过度跨越当前已提交堆页会触发访问异常，公开环境中约 `0x800` 字节是安全上限；
- readline 以换行结束，任何运行时地址中出现 `0x0a` 或 `0x0d` 都会截断载荷，这种实例应重连而不是强行继续。

### 为什么不能直接启动 shell

尝试：

```text
WinExec("cmd /c type flag.txt")
```

存在两个问题：

1. `WinExec` 异步返回，父进程很快从不完整 ROP 链崩溃并关闭连接；
2. 子进程的标准输出并不等于当前 pewpew socket。

即使让父线程停在死循环，`cmd.exe` 仍无法把 flag 返回给攻击者。因此应始终留在当前进程，调用 `NtOpenFile`、`NtReadFile` 和已有 WriteFile wrapper。

### 从 PEB 取得随机当前目录句柄

程序每次运行的目录形如：

```text
\Device\HarddiskVolumeX\Users\...\Temp\...\pewpew_<random>\
```

随机目录不能跨连接硬编码。Windows 进程自身已经在 PEB 链中保存当前目录信息：

```text
PEB
  -> RTL_USER_PROCESS_PARAMETERS
       -> CurrentDirectory
            -> Handle
```

ROP 链先调用 `NtQueryInformationProcess` 获取 PEB 地址，再用 `NtReadVirtualMemory` 分三次做 8 字节解引用：

```text
PEB -> ProcessParameters
ProcessParameters -> CurrentDirectory
CurrentDirectory -> directory HANDLE
```

把该句柄写入 `OBJECT_ATTRIBUTES.RootDirectory`，文件名只需使用相对路径：

```text
flag.txt
```

于是：

```c
NtOpenFile(
    &flag_handle,
    FILE_GENERIC_READ,
    &object_attributes_with_current_dir_handle,
    &io_status,
    FILE_SHARE_READ,
    FILE_NON_DIRECTORY_FILE
);
```

不再需要拼接随机 NT 路径。

### 单连接 ROP 调用链

完整调用顺序为：

```text
NtQueryInformationProcess
  -> 取得 PEB
NtReadVirtualMemory × 3
  -> PEB -> ProcessParameters -> CurrentDirectory HANDLE
NtOpenFile("flag.txt", RootDirectory=current_dir_handle)
NtReadFile(flag_handle, ...)
WriteFile-wrapper(socket, buffer, ...)
```

`NtOpenFile` 会把新句柄写到攻击者指定的内存地址。可以把输出位置直接安排在下一段 ROP 将要弹入参数寄存器的栈槽中，使句柄无需额外 gadget 就流入 `NtReadFile`。

成功连接会收到类似：

```text
r3ctf{e5537f8a-78e4-9f29-714c-fba0fa9774e6}
```

该 flag 由平台按队伍动态派生，正文中的值只用于说明验证形态。公开解法的对象逆向和实际 ROP 调试记录可参阅 [hax1ng 的 pewpew 题解](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/pewpew.md)；正文已经保留 UAF、泄漏、任意读写、`NtContinue`、PEB 链和单连接文件读取的完整主线。

## 方法总结

- 核心技巧：释放 Pilot 后继续通过悬空指针调用 circuit，再让同尺寸 Scout 复用其堆块；残留字段同时泄漏 EXE 基址与堆地址，伪 vtable 进一步提供任意读写。
- 识别信号：Windows 菜单程序若让不同类型共享固定大小、释放后仍保留全局对象指针，并允许新对象查看未清零内存，应优先检查 LFH 复用和 UAF。
- 复用要点：`NtContinue` 是从对象控制转向完整寄存器控制的稳定桥梁；随机工作目录可通过 PEB 的 `CurrentDirectory.Handle` 解决；当子进程不能继承网络输出时，应在原进程中相对打开、读取并使用已有 socket 写函数回传。
