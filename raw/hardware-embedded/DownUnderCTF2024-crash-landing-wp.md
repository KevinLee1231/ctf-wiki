# Crash Landing

## 题目简述

题目提供一份运行在 Cortex-M3 上的固件。程序从 UART 读取解锁码，并逐字节与秘密数据比较；奇怪的是，比较过程中会访问 `0xDEAD0000` 和 `0xBEEF0000` 两段并不存在的内存。目标是理解这些非法访问如何被固件利用，并还原正确输入。

## 解题过程

主循环的比较逻辑可以概括为：

```c
uint8_t secret_char = read_memmaped_u8(&flag_ptr[j]);
uint8_t sbox_mapping = read_memmaped_u8(&sbox_ptr[unlock_code_buf[j]]);

if (secret_char != sbox_mapping) {
    is_valid = false;
}
```

其中 `flag_ptr = 0xDEAD0000`，`sbox_ptr = 0xBEEF0000`。这两段地址未映射，正常读取会触发精确总线错误并进入 HardFault。题目附带的 QEMU 补丁把 Netduino 2 的 `ignore_memory_transaction_failures` 从 `true` 改成 `false`，正是为了让这些访问真的产生异常。

异常入口先根据 `LR` 的异常返回位选择 MSP 或 PSP，再把异常栈帧交给 C 处理函数。处理函数读取 CFSR 和 BFAR：只有在故障精确且 BFAR 有效时，才根据故障地址模拟一次读取。

```c
if ((bfar & 0xFFFF0000) == (uint32_t)flag_ptr) {
    frame->r3 = flag[bfar & 0xFFFF];
    frame->return_address += 2;
    return;
}

if ((bfar & 0xFFFF0000) == (uint32_t)sbox_ptr) {
    frame->r3 = (39 * (bfar & 0xFF) + 7) % 256;
    frame->return_address += 2;
    return;
}
```

这里修改的是异常栈帧中的 `r3` 和返回地址：`r3` 被填入虚拟内存的“读取结果”，而 PC 增加 2 字节跳过触发异常的 Thumb `ldr`。异常返回后，调用者就像完成了一次普通读取。

因此 `0xDEAD0000+i` 返回密文字节 $c_i$，`0xBEEF0000+x` 返回仿射映射：

$c=(39x+7)\bmod256$

因为 $39^{-1}\equiv151\pmod{256}$，逐字节逆变换即可恢复输入：

$x=151(c-7)\bmod256$

对应的解码代码为：

```python
ciphertext = [
    99, 250, 60, 211, 177, 196, 162, 204, 162, 204, 128, 162,
    243, 23, 128, 240, 101, 128, 162, 204, 162, 162, 206, 23,
    128, 67, 243, 26, 143, 204, 165, 206, 165, 182, 67, 18,
]

plaintext = bytes((151 * (value - 7)) % 256 for value in ciphertext)
print(plaintext.decode())
```

输出为：

```text
DUCTF{m3m3_m4p_or_m3mmap_d45832a29d}
```

## 方法总结

本题把 Cortex-M 的精确总线错误、BFAR 和异常栈帧组合成了软件实现的“内存映射设备”。逆向时不能在看到无效地址后便认定程序无法运行，而要继续检查 HardFault 是否修复寄存器与返回地址。确定两个虚拟地址区间的语义后，剩余部分就是求模 $256$ 的仿射逆变换。
