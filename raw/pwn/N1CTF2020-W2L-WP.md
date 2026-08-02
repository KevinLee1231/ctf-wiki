# N1CTF 2020 W2L Writeup

## 题目简述

W2L 是 Linux 内核利用题，核心漏洞为 `CVE-2017-7308`：AF_PACKET 的环形缓冲区计算错误可造成内核堆越界写。预期环境启用 KASLR、SMEP 和 SMAP，要求先通过相邻的 `user_key_payload` 泄露内核地址，再使用公开控制流原语提权。赛时启动配置意外关闭 SMEP/SMAP，且文件权限还产生过更简单的非预期路径。

## 解题过程

### 触发 packet socket 越界写

创建 `AF_PACKET` socket，配置 `PACKET_RX_RING` 和特制的 block/frame 参数。内核在计算 `tp_sizeof_priv`、block 边界与对齐时发生整数关系错误，使后续包处理可写到环形缓冲区对象之外。公开的 CVE exploit 已能把该越界转化为相邻堆对象字段覆盖。

题目预期不是原样执行旧 exploit，而是处理三项保护：

```text
KASLR：未知内核基址
SMEP：禁止内核执行用户页代码
SMAP：限制内核直接访问用户页数据
```

### 用弹性结构泄露内核基址

大量喷射 `user_key_payload`，让 key 对象与 packet ring 相关对象相邻。利用越界写放大 key payload 的长度字段，再通过正常的 `keyctl_read()` 把超出原分配范围的相邻内核内存复制到用户空间。

从泄露数据中寻找落在内核文本区或已知全局对象的指针：

```text
kernel_base = leaked_pointer - symbol_offset
```

这种方法把“可越界写”先转成“可控长度的内核到用户复制”，官方材料称其为可泄露的弹性结构。只有在确认指针类型和符号偏移后，才应把它用作 KASLR 基址。

### 完成控制流劫持

取得内核基址后，按 CVE-2017-7308 的公开利用链覆盖定时器或回调结构，把执行流导向内核 ROP。ROP 调用 `prepare_kernel_cred(0)` 与 `commit_creds()`，必要时先处理 SMEP/SMAP，再安全返回用户态并启动 shell。

仓库说明引用的 [ELOISE 论文](https://zplin.me/ELOISE.pdf) 解释了自动寻找“长度可被扩大、随后能复制到用户态”的内核对象；出题人的 [泄露 PoC](https://github.com/Markakd/n1ctf2020_W2L/blob/main/leak.c) 则给出本题具体喷射与读取方式。关键思想已在上文展开，链接用于核对实现细节。

### 赛时非预期解

实际启动参数关闭了 SMEP/SMAP，传统 ret2usr 即可工作。另有实例把 `/bin` 相关文件权限配置错误，普通用户能替换或伪造系统稍后调用的 `umount`，直接读取 `/root/flag`。这两条都属于部署偏差，不能代替预期的泄露与内核 ROP 分析。

## 方法总结

内核题必须分别验证漏洞、信息泄露和控制流三层。开启 KASLR 时，任何固定地址 exploit 都不完整；开启 SMEP/SMAP 时，直接跳用户态 shellcode 也不成立。与此同时，比赛镜像的权限和启动参数是实际攻击面，发现非预期解时应记录其环境前提，而不是反推成源码设计。
