# NFT Trove

## 题目简述

目标是 QEMU 中运行的 ARM926EJ-S/FreeRTOS 服务。用户提交 NFT 长度和十六进制数据，flag 预先通过 heapWrite 放进堆。长度读入有符号 int，只拒绝 0 和大于上限的值；负数传给 size_t 后变成巨大无符号数，引出 malloc 失败、空指针写和从低地址开始的超长 memcpy，最终可在复位向量附近放置 ARM shellcode。

## 解题过程

输入 -10 可以绕过：

~~~c
if (n == 0) reject();
if (n > RECV_BUFFER_LEN - 4) reject();
heapWrite(n, byteData);
~~~

进入 heapWrite 后，n 被转换为 size_t。pvPortMalloc(n + 4) 请求巨大内存并返回 0，但返回值未检查：

~~~c
dest = pvPortMalloc(n + 4);
*((int *)dest) = 0xABADBABE;
memcpy(dest + 4, data, n);
~~~

于是攻击数据从地址 4 开始覆盖。超长复制最终触发异常/复位，而低地址复位向量区域已经包含攻击者字节，CPU 会转入这些 ARM 指令。利用脚本先发 -10，再把 shellcode 作为十六进制行发送。

仓库保留的 264 字节 shellcode经 ARM 反汇编可确认完成三件事：

1. 在栈上拼出并加载 UART0 MMIO 地址 0x101f1000。
2. 从地址 0x10000 起逐字节扫描内存，寻找 ASCII 前缀 map。
3. 命中后把后续 100 字节逐字节写入 UART0 数据寄存器。

flag 在启动时已经被复制到堆中，因此扫描最终命中并输出：

~~~text
maple{y0u_c4nt_ju5t_g0_4nd_c0py_my_nft}
~~~

## 方法总结

这条链同时依赖有符号/无符号转换、未检查分配失败、空指针附近可写、嵌入式异常向量和 MMIO 输出。分析固件 pwn 时不能照搬桌面 Linux 的“空指针必崩溃”假设，应结合芯片内存映射、复位行为和串口寄存器验证。所有分配结果都必须检查，长度在转换为 size_t 前应验证为严格正数并做加法溢出检查。
