# BabyCOM

## 题目简述

题目运行在 Windows Server 2025 虚拟机中。低权限用户可以调用以 `LocalSystem` 身份运行的 COM 服务 `VaultSvc`，flag 位于只有管理员或 SYSTEM 才能直接读取的第二块裸磁盘 `\\.\PhysicalDrive1`。

附件中的 `vaultsvc.idl` 定义了 `IVaultService`，关键接口包括：

```text
OpenSession / CloseSession
StageVault / AppendVaultData / ResizeVault / CommitVault
QueryVaultInfo
```

每个接口都接收 `requesterVersion`，并可通过 `pServiceVersion` 返回服务版本。逆向 `vaultsvc.exe` 可确认，版本检查会以 SYSTEM 权限读取固定路径：

```text
C:\ProgramData\Vault\Version.txt
```

若请求方版本不匹配，文件内容会经 `pServiceVersion` 返回。正常情况下只能读到版本号；若能把服务内存中的路径字符串改成 `\\.\PhysicalDrive1`，这个版本检查就会变成一个 SYSTEM 权限的任意文件读取代理。

题目并不要求获得 SYSTEM 代码执行。预期路线是利用 `CommitVault` 与 `ResizeVault` 之间的竞态产生一次受控越界写，定点改写版本文件路径，再通过版本不匹配取回 flag。

## 解题过程

### 确认版本检查是可利用的数据出口

向 `OpenSession` 传入错误版本时，服务会返回 `HRESULT 0x8007051A`，同时把实际版本文件的内容写入 `pServiceVersion`。读取逻辑还包含面向无缓冲设备的页对齐回退路径，因此它不仅能读取普通文本，也能处理裸磁盘设备。

直接把 `Version.txt` 替换为符号链接不可行：`C:\ProgramData\Vault` 只允许 SYSTEM 和管理员修改，低权限用户也没有创建符号链接所需的权限。因此需要改变服务进程自身使用的路径字符串。

### `CommitVault` 与 `ResizeVault` 的 TOCTOU

每个 vault 对象保存一块数据、当前偏移和声明长度 `declaredLen`。`CommitVault` 的关键顺序可简化为：

```c
len = vault->declaredLen;
if (len > 128)
    return ERROR_INVALID_SIZE;

write_stage_file(...);               // 较慢的磁盘 I/O
memcpy(stack_buffer, vault->data, vault->declaredLen);
lock_vault_table();                  // 危险拷贝之后才加锁
```

`ResizeVault` 能把 `declaredLen` 改到 `0x1000`，而且没有和上述检查、拷贝共享同一把锁。于是可以让两个线程并发执行：

```text
线程 A：CommitVault 看到 declaredLen = 1，通过检查并进入磁盘 I/O
线程 B：ResizeVault 把 declaredLen 改成 0x100
线程 A：按新的 0x100 长度向栈缓冲区复制
```

这就是检查时与使用时不一致的 TOCTOU。磁盘写入扩大了竞态窗口，实际不需要高频暴力尝试。

### 把越界写转成 write-what-where

栈缓冲区起始于 `rsp+0x50`。在相对缓冲区偏移 `0x80` 的位置保存着一个目标指针，函数稍后会把经过单字节 XOR 处理的数据复制到该指针指向的位置。

把竞态后的长度设为 `0x100`，即可覆盖这个指针，但仍远未触及位于约 `0x320` 偏移处的栈 cookie。于是函数会在 cookie 校验前执行：

```c
memcpy(attacker_selected_address, attacker_selected_data, copy_length);
```

利用目标选择为模块内的可写路径字符串：

```text
target = module_base + 0x26000
value  = L"\\\\.\\PhysicalDrive1"
```

这条路线只修改数据，不劫持控制流，也不需要绕过 CFG、DEP 或栈 cookie。

### 泄漏 ASLR 基址

`QueryVaultInfo` 返回的缓冲区大于实际初始化的数据，尾部会带出栈残留。其中包含指向 `vaultsvc.exe` 的代码指针。公开解法使用：

```python
module_base = leaked_code_pointer - 0x2610
path_address = module_base + 0x26000
```

偏移与题目附件中的具体二进制绑定；换版本后必须重新在反汇编中定位泄漏返回点和路径字符串 RVA。

### 处理提交数据的单字节 XOR

`CommitVault` 的第二次复制会对数据做单字节 XOR，密钥由当前秒附近的线性同余状态导出。服务也会把处理后的已知数据作为提交结果返回，因此可以先提交已知字节，直接恢复当次 XOR key：

```python
xor_key = known_plaintext_byte ^ returned_byte
encoded_path = bytes(byte ^ xor_key for byte in utf16_device_path)
```

每轮竞态前重新恢复 key，再把预异或的 UTF-16LE 设备路径和 `path_address` 放入溢出数据。即使竞态没有命中，也不会用过期 key 破坏全局路径。

### 完整利用顺序

利用程序可用 MIDL 生成的代理声明或 C# COM interop 调用 `IVaultService`。主线为：

1. 创建会话和 vault，通过 `QueryVaultInfo` 取得代码指针，计算模块基址。
2. 提交一个已知字节，从返回值恢复当前 XOR key。
3. 构造至少 `0x88` 字节的数据：
   - 前部放置预异或后的 `L"\\\\.\\PhysicalDrive1\0"`；
   - 偏移 `0x80` 处放置 `module_base + 0x26000`。
4. 先以很小的 `declaredLen` 调用 `StageVault`，再用 `AppendVaultData` 放入完整载荷。
5. 多个线程并发调用 `ResizeVault(vaultId, 0x100)`，主线程调用 `CommitVault`。
6. 竞态命中后，再以错误版本调用 `OpenSession`。
7. 服务以 SYSTEM 权限打开已经被改写的设备路径，并通过 `pServiceVersion` 返回磁盘开头的 flag。

一次成功运行的关键结果形态为：

```text
module base  -> 0x00007ff7........
path address -> module base + 0x26000
pServiceVersion -> r3ctf{intended-flag-extraction-without-code-exec}
```

公开解法中的具体逆向结果和竞态实现可参阅 [hax1ng 的 BabyCOM 题解](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/BabyCOM.md)。正文已经保留利用所需的接口、竞态顺序、关键栈偏移、地址计算与 XOR 处理，链接仅用于核对原始分析记录。

## 方法总结

- 核心技巧：利用 `CommitVault` 检查长度后才加锁、`ResizeVault` 又能无锁改长度的 TOCTOU，把受限提交变成栈越界写；再覆盖函数稍后使用的目标指针，获得 cookie 检查前的 write-what-where。
- 识别信号：当 COM/RPC 服务以高权限读取文件并把内容返回给低权限调用方时，路径字符串本身就是高价值攻击目标；存在“检查—慢 I/O—使用”顺序且共享状态可被另一接口修改时，应优先检查竞态。
- 复用要点：先用独立泄漏处理 ASLR；只覆盖 cookie 之前会被使用的数据指针；对时间相关编码每轮重新校准；若最终目标只是让高权限服务替自己读取受保护对象，不必把一次任意写扩展成完整代码执行。
