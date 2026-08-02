# N1CTF 2021 - hacker vs hacker

## 题目简述

附件是一份包含大量 TCP 会话的 PCAP。客户端向 9999 端口发送经过汇编混淆的 AMD64 shellcode；每段 shellcode 都测试 flag 的某个位置是否等于某个字符，但网络中没有直接回显比较结果。正确与否被编码在连接由哪一端先关闭这一 TCP 状态中。

决定性信息藏在 FIN 时序这一协议隐蔽信道里，因此归入 Stego；提取出正确会话后，还需模拟 shellcode 才能恢复字符位置和值。

## 解题过程

### 理解 TCP 状态侧信道

去混淆后的每段载荷等价于：

```c
fd = open("meow", O_RDONLY);
read(fd, flag_buf, 0x120);
if (flag_buf[position] == candidate) {
    while (1) { }
} else {
    *(volatile int *)0 = 0;  // 崩溃
}
```

候选错误时，服务端进程崩溃并主动关闭连接，所以服务端先发 FIN。候选正确时，shellcode 进入死循环，客户端超时后主动断开，所以客户端先发 FIN，服务端的 FIN 成为该流中的第二个 FIN。

因此只需保留“第二个 FIN 由受害服务端发出”的 TCP 流。官方脚本用 `tshark` 同时导出目标端口、FIN 标志、TCP payload 和 stream id：

```sh
tshark -r cap.pcapng -T fields -E separator=, \
  -e tcp.dstport -e tcp.flags.fin -e tcp.payload -e tcp.stream \
  -E header=n 'tcp.port==9999'
```

对每个 `tcp.stream` 建立状态机：客户端发往 9999 端口且长度超过 1600 字节的 payload 是待分析 shellcode；计数两个 FIN，只有第二个 FIN 的目标不是 9999 端口时才保存该 payload。

```python
if not is_fin and dst_port == 9999 and len(payload) > 1600:
    state.shellcode = payload
if is_fin:
    state.fin_count += 1
    if state.fin_count == 2 and dst_port != 9999:
        successful_payloads.append(state.shellcode)
```

### 用 Miasm 提取比较位置和字符

无需完整反混淆每段 shellcode。把代码映射到固定地址并用 Miasm 执行，只需挂钩两个语义点：

1. 遇到 `read` 系统调用时，`RSI` 是 flag 缓冲区地址，`RDX` 是长度。记录基址、填入零数据，并对该区域设置读断点。
2. shellcode第一次读取 flag 缓冲区时，访问地址减基址就是 `position`；当时寄存器中的立即数候选就是 `candidate`。

官方 `t.py` 的关键逻辑如下：

```python
def handle_syscall(jit):
    if jit.cpu.EAX == 0:             # read
        flag_base = jit.cpu.RSI
        jit.vm.add_memory_breakpoint(flag_base, jit.cpu.RDX, PAGE_READ)
        jit.vm.set_mem(flag_base, b"\x00" * jit.cpu.RDX)
    jit.cpu.set_exception(0)
    return True

def handle_flag_read(jit):
    address, _ = jit.vm.get_memory_read()[0]
    position = address - flag_base
    candidate = chr(jit.cpu.RAX)
    return False
```

并行模拟所有成功流，把 `(position, candidate)` 写回同一个字节数组，就能按位置重组 flag。仓库没有附带原始 PCAP 或官方脚本运行输出，因此现有材料不足以给出最终 flag。

### 非预期的时间爆破

官方 README 还记录了一个更直接的非预期解：把 shellcode封装成可执行文件，先用 `strace` 找到目标文件名 `meow`；然后为目标位置枚举字母、数字等候选字符。错误候选会立即崩溃，正确候选会运行超过约 5 秒，通过本地运行时间即可判断。这仍然利用同一个“崩溃与死循环”侧信道，只是绕过了 PCAP 中的 FIN 分类。

## 方法总结

网络取证不能只看 payload，连接状态和关闭顺序也可能承载数据。先用 `tcp.stream` 重建每条会话，再依据双方 FIN 的方向筛出正确候选，最后用系统调用与内存访问挂钩提取最小语义，比完整去混淆上百段 shellcode 更稳妥。
