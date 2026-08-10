# Patchme

## 题目简述

程序自身包含 `gets` 栈溢出和 `printf(format)` 格式化字符串漏洞，题目要求在不改变文件大小及主要业务逻辑的前提下修复二进制。程序还使用反调试与 SMC 自修改代码保护真正的校验和 flag 输出逻辑，因此也可以绕过反调试后直接还原隐藏代码。

## 解题过程

### 路线一：修复两个漏洞

主函数的危险逻辑可以概括为：

```c
char format[24];
gets(format);
printf(format);
```

第一处应改成限制长度的输入，例如：

```c
scanf("%23s", format);
```

第二处应使用固定格式串：

```c
printf("%s", format);
```

原调用点空间不足以直接容纳所有新指令，可以把新增汇编写入 `.eh_frame` 等未参与正常执行的空隙，再从原调用点跳转过去，执行后跳回。写入格式串地址时还要保持 x86-64 ABI 所需的栈对齐。`scanf` 使用过的 `%23s` 字符串随后也可复用于 `printf` 的格式参数，从而减少新增数据。

不能把主函数中看似无用的初始化代码随意删掉来腾空间：题目明确要求不改变其他逻辑，而且 `.text` 中部分代码属于 SMC 解密后的结果，直接覆盖会破坏隐藏流程。完成 patch 后，程序会运行内部检查并输出 flag。

### 路线二：还原 SMC 代码

隐藏函数先读取 `/proc/self/status` 检查调试状态，随后调用 `mprotect` 把目标内存改为可读、可写、可执行，并把从 `0x14C6` 开始的 `0x960` 字节逐字节异或 `0x66`：

```python
for address in range(0x14C6, 0x14C6 + 0x960):
    original = ida_bytes.get_byte(address)
    ida_bytes.patch_byte(address, original ^ 0x66)
```

调试时可以反转反调试跳转或临时跳过 `exit(0)`，等解密完成后把该区域标记为代码。隐藏逻辑最终把两组连续的 47 字节数据异或输出。按真实内存布局打包常量即可直接复现：

```python
import struct

left = struct.pack(
    "<5Qihb",
    0x5416D999808A28FA,
    0x588505094953B563,
    0xCE8CF3A0DC669097,
    0x4C5CF3E854F44CBD,
    0xD144E49916678331,
    -631149652,
    -17456,
    85,
)
right = struct.pack(
    "<5Qihb",
    0x3B4FA2FCEDEB4F92,
    0x07E45A6C3B67EA16,
    0xAFE1ACC8BF12D0E7,
    0x132EC3B7269138CE,
    0x8E2197EB7311E643,
    -1370223935,
    -13899,
    40,
)

print(bytes(a ^ b for a, b in zip(left, right)).decode())
```

输出为：

```text
hgame{You_4re_a_p@tch_master_0r_reverse_ma5ter}
```

## 方法总结

二进制 patch 不只是把危险函数名替换掉，还要处理参数准备、指令长度、跳板空间、栈对齐和文件大小约束。面对 SMC 时，应先定位 `mprotect`、解密循环和反调试分支；让程序完成自解密后再分析，通常比静态阅读密文字节更直接。
