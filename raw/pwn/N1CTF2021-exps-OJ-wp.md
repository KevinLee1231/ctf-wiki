# N1CTF 2021 - exp's OJ

## 题目简述

题目要求连续上传三段仅由数字与英文字母范围字符构成、长度不超过 1400 字节的 AMD64 shellcode，分别完成排序、Base64 编码和带噪序列对齐。runner 把代码复制到 RWX 内存后启用 strict seccomp，只允许 `read/write/_exit/sigreturn`，因此三关都必须在短小的纯 shellcode 中完成。

官方提供了自制的字母数字 shellcode 编码器：先用简单 XOR 压缩减少需要转换的字节，再生成可打印的自解码代码。

## 解题过程

### 构建可上传的 shellcode

服务端把 runner 模板复制到临时文件，并在固定偏移 `0x2c0` 覆盖选手代码。runner 自己申请 `0x210000` 字节 RWX 内存、复制前 `0x1000` 字节、启用 `SECCOMP_MODE_STRICT` 后跳转执行。

官方 `run.py` 的构建过程是：

```python
gcc -Os -nostdlib -nodefaultlibs -fPIC challN.c
```

随后用 `pyelftools` 取出 ELF `.text`，在开头补一条跳到 `_start` 的短跳转，再调用：

```python
encoder_with_xor_compress(shellcode=code, base_reg="rdx", offset=0)
```

编码器先寻找可用字母数字立即数对，对原 shellcode 做分块 XOR 压缩，再生成自修改解码器。最终三段编码结果均需小于 `MAX_CODE_SIZE=1400`。服务端名义上检查字母数字，实际 `A <= ch <= z` 还放行了 ASCII 中间的少量标点，但官方编码器不依赖这个额外范围。

### 第一关：排序

提示写“1000 个数”，源码实际发送的是 `0x1000` 个 32 位无符号整数。shellcode逐个读入栈数组，原地升序排序后写回全部 4096 项。官方解法使用最短的冒泡排序：

```c
for (int i = 0x1000 - 1; i > 0; i--)
    for (int j = 0; j < i; j++)
        if (numbers[j] > numbers[j + 1])
            swap(numbers[j], numbers[j + 1]);
```

服务端用 `qsort` 生成标准答案并逐字节比较。时间限制很紧，但编译后的简单循环加 `-Os` 仍能满足题目环境。

### 第二关：Base64

服务端输入 256 个随机字节，要求恰好输出 344 字节标准 Base64。前 255 字节组成 85 组三字节，最后一个字节产生两个字符和 `==`：

```c
for (int i = 0; i < 85; i++) {
    // 按 6 bit 查 "A-Z a-z 0-9 + /" 表并 write 四次
}
write_char(table[data[255] >> 2]);
write_char(table[(data[255] & 3) << 4]);
write_char('=');
write_char('=');
```

把 64 字节码表放进 `.text` 自定义节，编码后可随 shellcode一起进入 runner。

### 第三关：局部序列对齐

服务端先随机生成 AES-128-CBC 的 key 与 IV，并加密 flag 主体的 32 字节。然后两次把这 32 个密文字节藏进各自 0x400 字节随机序列：

- 每个真实字节都叠加 $[-5,5]$ 的随机偏移；
- 两个真实字节之间随机插入噪声字节；
- 每条序列包含 6 到 16 个插入噪声。

shellcode需要从两条序列中找出共同的 32 字节骨架。官方 `chall3.c` 实现了类似 Smith-Waterman 的局部对齐：两个字节差的绝对值小于 10 时给正分，差越小得分越高；横向/纵向移动施加 gap 惩罚；每个 DP 单元低两位保存回溯方向，其余位保存分数。找到全局最高单元后回溯，遇到对角转移就从第二条序列取出一个字节，直到得到 32 字节。

服务端随后公开 Base64 编码的 key、IV、shellcode输出，以及第二条序列使用的 32 个偏移。客户端逐字节消去噪声并按 256 取模：

```python
ciphertext = bytes((value - int(delta)) % 256
                   for value, delta in zip(recovered, offsets))
plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
```

仓库部署文件给出的最终结果是：

```text
n1ctf{hav3_Fun_wIth_she1lc0de_Enc0der_}
```

## 方法总结

本题重点是受限 shellcode工程，而非内存破坏：先写不依赖 libc 的最小 C/汇编算法，再抽取 `.text`，最后用自解码器满足字符集。第三关把动态规划方向压进 DP 分数的低两位，在有限代码尺寸下同时保存分数和回溯信息，是最值得复用的实现技巧。
