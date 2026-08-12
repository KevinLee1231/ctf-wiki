# DownUnderCTF 2022 just-in-kernel Writeup

## 题目简述

内核模块创建可写的 `/proc/challenge`，把用户提交的简易指令解析成 `mve/add/cmp/jmp`，在 `PAGE_KERNEL_EXEC` 的 `vmalloc` 页面中生成 x86-64 机器码并直接以内核权限调用。QEMU 启用了 SMEP、SMAP，但启动参数包含 `nokaslr`。

JIT 表面只提供四个寄存器和少量操作，漏洞却允许跳进 `movabs` 的立即数区域，把本应是“数据”的 8 字节当作任意机器码执行。

## 解题过程

### 把 `mve` 立即数变成代码槽

一条 `mve a, value` 被编译为 10 字节：

```text
48 b8 <8-byte immediate>
```

若最后提交 `jmp 2`，编译器会跳到 JIT 页面基址加 2，正好落在第一条 `mve` 的立即数开头。每个立即数布置为：

```text
<最多 6 字节有效指令> eb 02
```

末尾的 `jmp short +2` 跳过下一条 `mve` 的 `48 b8` 前缀，继续执行下一块立即数。重复这一结构，就能把许多 6 字节片段串成任意内核 payload，同时仍通过 JIT 的语法检查。

```python
def slot(code6):
    raw = code6.ljust(6, b'\x90') + b'\xeb\x02'
    return f"mve a {u64(raw)}\n"

payload = ''.join(slot(chunk) for chunk in chunks) + 'jmp 2'
```

### 提权并安全返回用户态

由于 KASLR 关闭，官方内核中的关键地址固定：

```text
prepare_kernel_cred = 0xffffffff81084d30
commit_creds        = 0xffffffff810848f0
```

payload 先在 `rbx` 中分段构造 64 位地址，执行：

```c
commit_creds(prepare_kernel_cred(NULL));
```

此时当前进程获得 root credential。为了在 SMEP/SMAP 开启时正确回到用户态，exploit 启动前保存 `CS`、`SS`、`RSP` 和 `RFLAGS`，在内核栈上布置用户态 `iretq` frame，并跳入固定的 `swapgs_restore_regs_and_return_to_usermode` 路径。frame 的用户态 RIP 指向本地 `shell()`。

编译静态 exploit 后，通过终端把 gzip+Base64 分块传入虚拟机，执行程序向 `/proc/challenge` 写入 JIT payload。回到用户态时 UID 已是 0，读取：

```text
DUCTF{1_gu355_iT_W4SnT_m3m0Ry_s4Fe_4Fter_4ll}
```

## 方法总结

限制源语言指令集并不等于限制最终机器码。JIT 必须保证控制流只能落在合法指令边界，不能跳入立即数或跨越编码边界；否则立即数就是可执行字节仓库。本题的 shellcode 位于内核可执行映射中，所以 SMEP 不会阻止它，SMEP/SMAP 主要影响最后如何安全返回用户态。
