# guess-what

## 题目简述

题目是一个两阶段猜测程序。第一阶段要求输入与硬编码字符串完全相等；第二阶段读取整数 `guess` 并检查

$$
guess=42\times9/guess+3.
$$

无论猜测正确与否，程序随后都会读取并输出 `flag.txt`，但需要先通过第一阶段才能到达该路径。

## 解题过程

反编译或查看字符串可直接得到第一阶段输入：

```text
nuh uh pls nolfjdl
```

注意 `fgets` 保留换行，远程使用 `sendline` 正好补上源码比较中的末尾 `\n`。

第二阶段在 C 整数除法语义下求解。取 `guess=21`：

$$
42\times9/21+3=378/21+3=18+3=21.
$$

因此完整交互为：

```python
from pwn import remote

io = remote("tjc.tf", 31478)
io.sendlineafter(b"thinking\n", b"nuh uh pls nolfjdl")
io.sendlineafter(b":\n", b"21")
print(io.recvall().decode())
```

程序打印祝贺信息和：

```text
tjctf{n3v3r_c0uld_r34d_y0ur_m1nd_8e6646a1}
```

## 方法总结

- 第一关是精确字符串比较，必须留意 `fgets` 是否把换行包含在缓冲区内。
- 第二关使用整数除法，不应把等式直接按实数二次方程处理；候选值需要按 C 的运算顺序回代。
- flag 输出并不受第二个 `if` 的成功分支控制，但第一阶段失败会提前 `return`，所以最短路径仍是提交硬编码字符串和任一安全非零整数；使用 21 可同时通过提示检查。
