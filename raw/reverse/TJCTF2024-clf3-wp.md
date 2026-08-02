# clf3

## 题目简述

题目给出一个带反调试、RDTSC 时序检查和自校验的 ELF。它递归执行 `fluxion`，在最深层用栈上 17 字节循环异或解密内嵌的第二个 ELF，然后直接 `execl(arr, "")`。第二阶段又实现一个极小字节码解释器，对内置字符串逐字节变换后输出。目标是先导出内嵌程序，再还原其输出。

## 解题过程

第一阶段不需要重写整个解密算法。反汇编 `fluxion` 最深层可看到：解密完成后，本来有一段逐字节 `printf("%i,\n", arr[i])` 的调试循环，但入口的短跳转 `EB 36` 越过它直接执行 `execl`。把文件偏移 `0x1341` 的位移从 `0x36` 改为 `0x2d`，跳转就落到输出循环的条件判断，从而打印全部 18232 个解密字节。

修改文件会触发自哈希。`main` 在文件偏移 `0x1b36` 用 `74 1b`（`je`）通过校验；把 `0x74` 改成 `0xeb`（无条件 `jmp`）即可继续。仓库中的 `solve/clf3` 正是只改这两个字节的版本。

```python
from pwn import process

stage1 = process("./clf3")
lines = stage1.recvall().decode().splitlines()

payload = bytes(int(line.rstrip(",")) for line in lines[1:-1])
open("stage2", "wb").write(payload)
```

第二阶段的 VM 先从 `stack[0]` 读出常量 2 到 `rax`，随后对字符串区逐字节执行 `inc_ptr; add`，使加密字符加 2。程序打印出的结果仍比真正 flag 每字节大 4；官方解法对前 35 个变换字符再减 4，未加密尾部保持原样：

```python
import os
from pwn import process

os.chmod("stage2", 0o755)
encoded = process("./stage2").recvall().decode().strip()
flag = "".join(chr(ord(c) - 4) if i < 35 else c
               for i, c in enumerate(encoded))
print(flag)
```

最终得到：

```text
tjctf{Not_2_c0mplic2ted1ffaa1191099q221}
```

## 方法总结

- 遇到自解密程序，优先寻找作者残留的输出路径；改一条短跳转通常比完整复刻栈相关密钥更稳。
- 修改控制流后必须同时处理自校验。本题补丁范围可精确锁定为两个字节，避免大面积 NOP 破坏栈与解密状态。
- 第一阶段输出的是另一份 ELF，不是最终文本；继续分析其 VM 指令流，才能解释为什么还要对输出减 4。
