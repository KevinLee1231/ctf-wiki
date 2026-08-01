# GlacierCTF 2025 writergate_warmup

## 题目简述

题目是 Zig 编写的菜单程序，标准输入 reader 使用栈上的 `read_buffer[0x100]`。选择 3 后，程序打开 `/flag.txt` 并创建另一个 file reader，却把同一个 `read_buffer` 传给它。文件读取会覆盖仍被 stdin reader 认为有效的缓冲数据；回到菜单循环后，程序把被覆盖的 flag 字节当作选项，并在错误消息中逐字节打印。

这是 reader 状态与底层缓冲区发生别名的内存安全问题，不需要破解用于正常输出的 XChaCha20-Poly1305。

## 解题过程

### 1. 确认缓冲区被两个 reader 共享

初始化代码为：

```zig
var read_buffer: [0x100]u8 = undefined;
var stdin = std.fs.File.stdin().reader(&read_buffer);
```

菜单选项 3 又执行：

```zig
const ff = try std.fs.openFileAbsolute("/flag.txt", .{ .mode = .read_only });
var file_reader = ff.reader(&read_buffer);
var encryptor = try Encrypt.init(&crypt_state, &file_reader.interface, &encryptor_buffer);
_ = try encryptor.interface.streamRemaining(&stdout.interface);
```

两个 reader 各自保存 cursor/end 等状态，但都指向同一片 256 字节内存。`file_reader` 读取 flag 时改变底层字节，却不会同步更新 `stdin` 的 cursor 和已缓冲长度。

### 2. 让 stdin 预先缓存足够多的字节

如果只发送 `3`，stdin 可能只拥有很短的有效区间，覆盖后没有足够字节供循环继续解析。因此一次发送选项 3 和超过 flag 长度的填充，但不要让客户端逐行等待：

```python
io.sendafter(b"> ", b"3" + b"A" * 80)
```

首次 `takeInt(u8, .big)` 取出字符 `3`，其余 `A` 留在 stdin reader 已记录的缓冲区间中。选项 3 处理 flag 时，同一底层数组被 flag 明文覆盖；加密器输出什么并不重要。

### 3. 从错误选项恢复明文

回到循环后，stdin 继续按旧的 cursor/end 取字节。除 `1`、`2`、`3`、空格、换行等特殊值外，每个字节都会进入：

```zig
else => |c| try stdout.interface.print("No such option \\x{x}\n", .{c})
```

收集所有 `No such option \xNN` 行，将十六进制的 `NN` 转回单字节，并剔除尚未被覆盖的 `A` 填充，即可拼出：

```text
gctf{FreeRoundOfFlagsInHonorOfRefactoringBugsLeftUnnoticed}
```

## 方法总结

本题提醒我们，内存别名不一定表现为越界访问：两个安全抽象可以各自满足类型要求，却因共享底层 slice、不同步内部状态而泄漏数据。复现的关键是先让 stdin 的逻辑有效区足够长，再用 flag 文件读取替换物理字节，最后借错误信息逐字节观察。修复方式是为文件 reader 使用独立缓冲区，或在复用前彻底丢弃并重新初始化原 reader 状态。
