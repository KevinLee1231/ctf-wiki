# MTE

## 题目简述

题目运行在 AArch64 QEMU TCG 中，启动参数明确启用 `mte=on`、`pauth-qarma3=on`、3 个 vCPU 和 multi-thread TCG。上传的 solver 以 `shell` 用户执行，flag 位于仅 root 可读的 `/root/flag.txt`；`server` 用户独占 `/dev/mte_driver`，SUID `/bin/client` 负责建立 Binder 与 Unix socket 通道后再降权。

利用链把三类机制串在一起：

1. QEMU PAC 实现中题目补丁引入的特殊签名路径；
2. QEMU MTE tag 生成序列可预测，以及 Binder 文件描述符传递；
3. `mte_driver.ko` 的释放后使用与 Unix `SCM_RIGHTS` 垃圾回收竞争。

仓库没有 solver 源码，但提供了 QEMU 补丁、完整 initramfs 和编译后的官方 solver。以下机制来自补丁、init 脚本、驱动反汇编及 solver 的阶段字符串；无法由这些证据确认的内部偏移不作猜测。

## 解题过程

### 1. PAC 特殊路径与 QARMA3 黑盒恢复

补丁只改动 `target/arm/tcg/pauth_helper.c` 的 `pauth_addpac()`：

```c
if (arm_current_el(env) != 0 &&
    (ptr & MAKE_64BIT_MASK(56, 8)) == 0xa500000000000000ULL) {
    ptr &= MAKE_64BIT_MASK(0, 56);
    param.tsz = 8;
    param.tbi = false;
}
```

也就是说，内核态对顶字节为 `0xa5` 的指针执行 `PACIA` 时，QEMU 先清除顶字节，并强制按 `tsz=8、tbi=false` 计算 PAC；这不是普通 Linux PAC 语义，而是一条可被驱动批量调用的特殊 oracle。

驱动会生成多组 pointer/modifier，执行 `PACIA` 后把签名结果的顶字节交给用户态。官方 solver 的 `pac_solver` 将接口明确标为 `qemu_pacia_signed_top`，用分阶段的 QARMA3 黑盒筛选、少量 nibble 分支和最终枚举恢复唯一候选，再把 PAC 结果提交给驱动。日志中的 `front`、`peel`、`final_enum`、`PAC candidate hi/lo` 都属于这一阶段；它不是简单爆破一个 8 位 PAC。

### 2. 恢复 MTE tag 序列

驱动初始化时循环执行 AArch64 `IRG`，记录 272 个 4 位 allocation tag，并在映射给服务端的共享区中准备 tag/PAC 挑战。官方 solver 先通过 client/server 协议取得若干 MTE 样本，再恢复 QEMU MTE 伪随机生成器的 LFSR 状态，预测后续 256 个 tag：

```text
[solver] mte samples
[solver] predicted boot tags
[solver] submit tags
```

驱动逐项比较提交值的低 4 位与初始化时保存的 tag。PAC 和 MTE 两组校验都通过后，后续 race IOCTL 才会被接受。因此，MTE/PAC 在这里是进入漏洞接口前的认证门槛，而不是产生最终内核写的那处漏洞。

### 3. 通过 Binder 取得驱动 FD

`/dev/mte_driver` 权限为 `0600 server:server`，shell 不能直接打开。server 同时充当 Binder context manager；SUID client 建立 Binder 事务和 `/tmp/mte-client.sock` 控制通道，并在降权前准备共享对象。solver 驱动 client 发送一组 Binder FDA 更新事务，从 Binder 返回事件中取出 server 持有的驱动 FD，再用 `SCM_RIGHTS` 桥接回自身。

这一段的结果只是获得可调用 oracle/race IOCTL 的 FD，并没有直接获得 root。官方二进制对应的检查点包括 `asking client to send FDA update sequence`、`server did not return driver fd` 和 `captured driver fd in solver process`。

### 4. 驱动中的 stale object

反汇编 `mte_driver.ko` 可以确认 race 对象来自 `kmalloc-96`：

1. 分配请求大小为 `0x48`，驱动检查实际 `ksize()` 必须是 `0x60`；
2. 前 `0x48` 字节填为 `0x42`，尾部 `0x18` 字节填入随机值并另存一份；
3. `RACE_FREE` 对全局对象执行 `kfree()`，但保留全局指针；
4. 后续检查仍从该 stale pointer 读取尾部随机值；若未被破坏，就向对象偏移 `0x40` 写入 qword `2`。

因此漏洞的核心是“释放后仍校验并写入”，可对恰好回收该 `kmalloc-96` 槽位的内核对象实施一次定值 UAF 写。PAC/MTE 校验只决定这些 IOCTL 是否可用。

### 5. Unix socket GC 竞争扩大为 `modprobe_path` 写

官方 solver 的 `racer_scc` 用大量 Unix socket 和 `SCM_RIGHTS` FD 列表制造垃圾回收竞争，默认目标列表含 251 个 FD。它配合：

- `BINDER_FREEZE` 暂停/恢复 server，固定关键时序；
- `sendmsg/recvmsg`、Unix GC flush 和 signal pipe 协调 race；
- `/dev/null`、pipe 与多线程 spray 回收 stale FD/对象；
- `MTE_DRIVER_IOCTL_RACE_ALLOC/FREE/RESET` 反复布置上述 `kmalloc-96` UAF；
- pipe pivot 检测 stale FD 是否已经别名到目标 pipe endpoint。

成功后，solver 把回收对象组织成 fake `skb`，利用 unlink 路径形成受约束内核写，并按 chunk 多次修改固定内核映像中的 `modprobe_path`。二进制日志直接给出了 `fake-skb unlink writes`、`modprobe_unlink_chunk`、目标物理地址换算以及每轮 repair/retry；这说明最终原语不是“覆盖 cred”，而是逐段改写 `modprobe_path`。

solver 把辅助脚本写到 `/tmp/a`，内容为：

```sh
#!/bin/sh
id > /tmp/modprobe_id
cat /root/flag.txt > /tmp/flag 2>/dev/null || cat /flag > /tmp/flag 2>/dev/null
chmod 666 /tmp/flag 2>/dev/null
```

随后执行未知格式文件 `/tmp/t` 触发内核的 modprobe helper，最后读取 `/tmp/flag`。

可概括为：

```text
QEMU PAC special path -> recover PAC answer
QEMU MTE tag sequence -> recover LFSR and predict tags
Binder FDA/SCM_RIGHTS -> obtain mte_driver fd
kmalloc-96 stale pointer -> fixed UAF write at +0x40
Unix SCM_RIGHTS GC race + pipe/devnull spray
  -> stale fd / fake skb unlink write
  -> overwrite modprobe_path
  -> helper copies /root/flag.txt to /tmp/flag
```

最终得到：

```text
SEKAI{w0w_n0w_y0u_pwn3d_3v3rything_b1nd3r_k3rn3l_pac_mte_all_byp4ss3d}
```

## 方法总结

本题必须把“认证门槛”和“内存破坏根因”分开：QARMA3 PAC oracle、MTE LFSR 与 Binder FD 传递用于取得 race 接口的使用资格；真正的内核漏洞是驱动释放对象后仍通过全局指针读取并写入。硬件内存安全机制不会自动修复对象生命周期错误。

只有二进制 solver 时，可靠的还原顺序是：先从启动脚本确定用户、设备权限与 flag 边界，再用补丁确定实现差异，用 initramfs 中的驱动反汇编确认分配尺寸和 UAF 写，最后用 solver 字符串确认喷射对象、触发目标和命令内容。这样可以写清主因果链，同时避免把诊断选项臆测成未经证明的具体结构偏移。
