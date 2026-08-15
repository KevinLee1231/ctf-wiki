# FreeNote

## 题目简述

FreeNote 是一个 amd64、No PIE、Partial RELRO、NX、Canary 的堆菜单程序，目标环境使用 glibc 2.32。全局 notes 数组保存 64 个指针；删除操作只执行 free(notes[i])，没有把槽位置空，因此同一指针仍可被 show 读取，也可被再次 free。

此外边界判断使用 i > 64 而不是 i >= 64，会额外接受索引 64。不过官方利用链不依赖这个越界，而是使用 UAF、A-B-A double free 和 safe-linking 下的 freelist poisoning。

## 解题过程

请求大小 0x0f 时，程序实际 malloc(size + 1)，落入 0x20 chunk。先分配 9 个同尺寸 chunk，再释放前 7 个填满 tcache。此时依次释放第 7、8、7 个 chunk，三个 chunk 会进入 fastbin，A-B-A 顺序绕过只检查链首的传统 fastbin double-free 防护。

delete 没有清空 notes 指针，所以 show 可以读出已释放 chunk 的 freelist 数据。glibc 2.32 对单链指针使用 safe-linking：

$$
\operatorname{encoded\_fd}
=\operatorname{target}\oplus(\operatorname{chunk\_address}\gg12).
$$

官方 exploit 先耗尽 tcache，再从 UAF 内容取得堆相关泄漏，用下面的变换伪造 freelist 指针：

~~~python
def protect_ptr(chunk_address, target):
    return target ^ (chunk_address >> 12)
~~~

第一阶段把分配目标导向固定地址 stdout@GOT。No PIE 使 GOT 地址已知，读取该槽得到 libc 中 stdout 的真实地址，从而计算目标 libc 2.32 的基址。

第二阶段换一个尺寸重复“填满 tcache → fastbin A-B-A double free → 耗尽 tcache → 写入编码 fd”的过程，把分配目标导向 __free_hook。随后：

1. 在重叠 __free_hook 的 chunk 中写入 system 地址。
2. 另一个正常 chunk 写入“/bin/sh”。
3. 删除该 chunk，实际调用 system("/bin/sh")。

利用成功后读取 flag：

~~~text
shellmates{ev3n_SafE_LINkinG_c4N'T_StOP_us!}
~~~

## 方法总结

堆指针置空必须发生在真正保存指针的槽位，单纯调用 free 不会消除 UAF。面对 glibc 2.32 的 safe-linking，直接写裸目标地址不会形成有效 freelist；必须先获得堆地址相关信息，再按 chunk_address>>12 编码。完整链条是“UAF 泄漏 → safe-linked freelist poisoning → libc 泄漏 → hook 覆盖”，并依赖题目给定 libc 版本中仍存在 __free_hook。
