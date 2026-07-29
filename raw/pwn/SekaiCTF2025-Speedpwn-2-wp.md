# speedpwn-2

## 题目简述

程序维护一块可调整大小的画布。像素访问使用：

```c
canvas[arg1 * size_y + arg2]
```

却没有检查两个坐标是否为负数或超出边界，因此可以相对画布堆块进行任意单字节越界读写。程序支持 resize，会反复 `free` 和 `malloc`；二进制未开启 PIE，GOT 与 BSS 地址固定。

官方解法利用越界修改 `tcache_perthread_struct`，让后续分配返回 BSS/GOT 附近，再分两阶段把 `free` 改成 `printf` 和 `system`。

## 解题过程

### 1. 把坐标换算成堆相对偏移

索引是有符号计算，负的 `arg1` 或 `arg2` 可以落到画布块之前。已知当前 `size_y` 后，可以为任意目标堆偏移求一组：

$$
\text{offset}=arg_1\cdot size_y+arg_2.
$$

以单字节操作逐步修改 tcache 元数据，避免一次写入宽度受限的问题。

### 2. 伪造 tcache 入口指向 BSS

glibc 的线程缓存保存每个 size class 的计数和链表头。官方利用修改对应 bin 的 `counts` 与 `entries`，把下一次同尺寸 `malloc` 的返回地址定向到：

```text
0x404070
```

该地址位于可写 BSS，并靠近 GOT。触发 resize 后，新“画布”实际上覆盖这片固定地址区域，于是可以通过正常画布写入改 GOT。

### 3. `free -> printf` 获取 libc 泄漏

第一次把：

```text
free@GOT = printf@PLT
```

并在待释放画布中放置格式串：

```text
%17$lp
```

下一次 resize 原本会执行 `free(canvas)`，现在变成：

```c
printf(canvas);
```

格式串从栈上泄露 libc 地址，结合配套 `libc.so.6` 的已知偏移计算基址。

### 4. `free -> system` 获取 shell

重新使用同一 tcache 定向原语，让分配再次落入 GOT 附近，写入：

```text
free@GOT = system
```

然后让画布内容为：

```text
/bin/sh\0
```

最后一次 resize 触发释放：

```c
system(canvas);
```

即得到 shell。

仓库正式挑战镜像中的 flag 为：

```text
SEKAI{L4st_y34rs_w4s_t0o_h4rd_1_h0p3_th1s_0n3_w4s_m0r3_s1mpl3!}
```

## 方法总结

本题的利用链很短，但每一步都复用了同一个 resize：

```text
负坐标 OOB
→ 改 tcache_perthread
→ malloc 到 BSS
→ GOT 劫持
→ free 触发目标函数
```

没有初始 libc 泄漏时，先把 `free` 变成 `printf` 建立格式化字符串泄漏，再改成 `system`，比直接猜测 libc 更稳。防护不仅要验证最终一维索引，还要在乘法前检查每个坐标及乘法溢出。
