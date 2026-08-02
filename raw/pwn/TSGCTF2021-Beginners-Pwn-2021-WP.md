# TSGCTF2021 Beginner's Pwn 2021 WP

## 题目简述

程序在栈上放置两个 64 字节数组，先从文件读取 flag，再用 `scanf` 接收猜测：

```c
char your_try[64] = {0};
char flag[64] = {0};

size_t length = fread(flag, 1, 64, fp);
scanf("%64s", your_try);

if (strncmp(your_try, flag, length) == 0) {
    win();
}
```

`win()` 会执行 `/bin/sh`。漏洞是 `%64s` 最多读取 64 个输入字节后，还会额外写入结尾的 `\0`；目标缓冲区却只有 64 字节，因此发生一字节越界写。

## 解题过程

在题目二进制的实际栈布局中，`flag` 紧邻 `your_try`，所以 `scanf` 自动追加的第 65 个字节会覆盖 `flag[0]`。攻击时通过原始网络连接发送 64 个空字节，再附加换行：

```python
io.sendline(b"\x00" * 64)
```

前 64 个空字节进入 `your_try`，自动终止符又把 `flag[0]` 清零。随后比较的两个字符串都从空字节开始：

```text
your_try[0] = 0
flag[0]     = 0
```

`strncmp` 在看到两边相同的字符串终止符后立即判定相等，不需要知道 flag 的其他内容。程序输出 `yes` 并调用：

```c
system("/bin/sh");
```

进入 shell 后读取 flag 文件，得到：

```text
TSGCTF{just_a_simple_off_by_one-chall_isnt_it}
```

这里必须使用能发送 NUL 字节的脚本或网络库；在普通终端中直接键入文本无法构造该输入。

## 方法总结

`scanf` 字段宽度表示最多读取多少个数据字符，不包含它随后补写的字符串终止符。对长度为 64 的数组，安全格式应是 `%63s`，或者改用能显式接收缓冲区长度的输入方式。利用还结合了 C 字符串的提前终止语义：只需把两个比较对象的首字节都变成零，`strncmp` 就不会继续检查后面的真实 flag。
