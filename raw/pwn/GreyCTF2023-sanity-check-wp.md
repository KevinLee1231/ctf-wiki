# GreyCTF 2023 Sanity Check

## 题目简述

题目在 QEMU 内核中提供 `/dev/dean` 字符设备。设备读取可泄露栈保护值、内核栈指针和代码指针，写入路径则能覆盖内核栈。利用者需要构造 KROP 提升为 root，返回用户态后再从内核内存中找出 flag。

## 解题过程

打开 `/dev/dean` 后执行两次 0x100 字节读取。官方利用在返回缓冲区偏移处解析：

```text
bptr[3] -> stack canary
bptr[4] -> kernel stack pointer
bptr[5] -> kernel text leak
```

将代码泄露按大页边界修正可得到内核基址，所有 gadget 和函数都用随题内核的固定偏移重定位。写入 payload 时先填充到溢出点并原样放回 canary，再布置：

```text
pop rdi; ret
0
prepare_kernel_cred
pop rdi; ret
kernel_stack
xchg rax, rdi; ...; ret
commit_creds
swapgs_restore_regs_and_return_to_usermode
```

`prepare_kernel_cred(0)` 返回 root 凭据指针，交换到 `rdi` 后传给 `commit_creds`。尾部填写保存过的用户态 `rip/cs/rflags/rsp/ss`，通过 swapgs/iret 路径安全返回 `win()`，获得 root shell。

flag 文件的普通读取路径仍受题目机制限制。仓库提供的 `find_flag.ko` 从一个 `kmalloc` 地址向上按页扫描内核内存，比较每页开头附近是否出现 `grey{`，找到后用 `printk` 输出；加载模块并查看 `dmesg` 可得：

```text
grey{ar3_y0u_st1ll_s4n3_4ft3r_0ur_s4n1ty_ch3ck?}
```

## 方法总结

内核栈溢出必须同时处理 canary、KASLR 和用户态返回现场。泄露原语一次解决这三项定位问题，KROP 再复用内核凭据 API 提权。得到 root 后仍要遵循题目最后的 flag 存储机制；本题用自定义模块扫描内核内存，而不是把“拿到 shell”误当作已经取得 flag。
