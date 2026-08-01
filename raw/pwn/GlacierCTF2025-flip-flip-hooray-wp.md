# GlacierCTF 2025 flip-flip-hooray

## 题目简述

题目在 ARM64 Linux 内核中加入了 666 号系统调用 `flipper(addr, bit)`，它会直接对给定内核地址的某一位执行异或。补丁本想用静态变量 `flips_left` 将权限限制为一次，但没有检查目标地址是否属于用户空间，也没有保护计数器自身。

因此先用唯一一次机会修改 `flips_left`，即可把“一次任意位翻转”升级为大量任意内核位翻转；随后改写稳定地址处的 `modprobe_path`，让内核以 root 执行攻击者脚本。

## 解题过程

### 1. 自举位翻转额度

补丁将计数器初始化为：

```c
static unsigned long flips_left = 0xDEADBADC0DE00001;
```

若第一次翻转普通目标，函数末尾会把计数器减到阈值 `0xDEADBADC0DE00000`，后续调用即被拒绝。但位翻转发生在减 1 之前，且 syscall 允许把计数器自身作为目标。ARM64 为小端序，在最高字节上翻转 bit 0，相当于先给整个 64 位数增加 $2^{56}$；函数末尾再减 1 后仍远高于拒绝阈值：

```c
syscall(666, FLIPS_LEFT_ADDR + 7, 0);
```

源码实例中 direct-map 别名地址为 `0xffffff8002381dc0`。第一次调用结束后仍拥有足够多的额度来改写任意字节。

### 2. 绕过内核地址随机化问题

普通 KASLR 会随机化内核映像的标准虚拟地址，但 ARM64 还存在将物理内存线性映射到内核虚拟地址的 direct map。题目采用的配置中：

```text
PAGE_OFFSET = 0xffffff8000000000
```

且内核在物理内存中的装载位置稳定，因此内核数据的 direct-map 别名也稳定。官方 exploit 使用：

```text
flips_left direct-map: 0xffffff8002381dc0
modprobe_path direct-map: 0xffffff80023fcbc0
```

这与 Google Project Zero 的 [Defeating KASLR by Doing Nothing at All](https://googleprojectzero.blogspot.com/2025/11/defeating-kaslr-by-doing-nothing-at-all.html) 所分析的前提一致：ARM64 线性映射基址由配置确定，若物理内核落点也不随机，攻击者可在无需传统 KASLR 泄漏的情况下定位其 direct-map 数据。这里已经概括了外链对利用所必需的结论。

### 3. 按位改写 `modprobe_path`

目标是把默认的 `/sbin/modprobe\0` 改为 `/tmp/a\0`。对旧字符串和目标缓冲区逐字节异或，枚举差分值中每一个置位的位置，再调用一次 `flipper(address+i, bit)`：

```c
for (size_t i = 0; i < len; i++) {
    unsigned char diff = old[i] ^ wanted[i];
    for (int bit = 0; bit < 8; bit++)
        if (diff & (1u << bit))
            syscall(666, MODPROBE_PATH_ADDR + i, bit);
}
```

目标缓冲区剩余部分必须写成 `\0`，否则内核会把旧路径尾部拼在新字符串后。

在 `/tmp/a` 写入可执行脚本，使其复制 `/flag.txt` 到普通用户可读的位置并 `chmod`。再执行一个具有未知二进制格式的文件，触发内核的 module autoload 路径；内核以 root 调用被改写的 `modprobe_path`，最后读取复制出的文件。

源码实例 flag 为：

```text
gctf{e6ec4d_1ncr3d1bl3_th4t_th1s_w0rks_b4842d}
```

## 方法总结

只有一次的强原语并不一定弱，尤其当原语能够修改自己的额度或策略状态。本题先完成自举，再利用 ARM64 direct map 的稳定地址解决 KASLR，最后按位实现任意字符串写并走 `modprobe_path` 提权。复现时应分别验证计数器确实增大、目标地址确为 direct-map 别名、路径尾部已清零，以及未知格式执行确实触发 helper；任何一个未经验证的假设都会让利用表现为无输出。
