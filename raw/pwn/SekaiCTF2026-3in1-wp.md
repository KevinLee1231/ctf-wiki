# 3in1

## 题目简述

题目把三个执行边界串在一起：输入的 JavaScript 先由 Ladybird 的 LibJS shell 执行，shell 位于 Linux 客体用户态，客体又运行在带有 `virtio-snd` 设备的 QEMU TCG 中。最终目标不是客体中的文件，而是在宿主执行 `/readflag sekai 3in1`。

题目固定了 Ladybird、Linux 6.1.176 和 QEMU 的提交版本。`ladybird_1.patch` 只构建 JS-only shell，并移除除 `gc()` 外的内部函数；`qemu_2.patch` 修复了另一处公开问题；真正为第三阶段重新引入漏洞的是 `qemu_1.patch`。官方脚本把后两段原生 ELF 以 Base64 嵌在 `solve.js` 中，形成完整的“一次输入、连续跨越三层边界”的利用链。

## 解题过程

### 1. LibJS：`Set.intersection` UAF

LibJS 的 `Set.prototype.intersection` 使用引用遍历底层 `Map`：

```cpp
for (auto const& element : *set) {
    auto in_other = TRY(call(vm, *other_record.has,
        other_record.set_object, element.key)).to_boolean();
    // 后续继续使用 element.key
}
```

问题在于 `other.has()` 是攻击者可控回调。回调中执行 `s.clear()` 会释放当前正在遍历的 HashTable，而 `element` 仍引用旧存储；回调返回后再次读取 `element.key`，形成 UAF。

官方脚本围绕这块被释放的存储建立两个基础原语：

1. `fakeobj(addr)` 在回调中用 `Uint8Array.setFromBase64()` 回收旧 bucket，把伪造的 tagged `Value` 作为交集元素返回；
2. `leak_ptr_try_for()` 在 `clear()` 和 `gc()` 后喷射 `WeakMap`，让其中的 `GC::Ptr<Cell>` 覆盖旧 `Value`，再把该指针按浮点数取回，得到 `addrOf`。

接着泄露 `MapIterator`，利用其内联属性与 `Map` 指针的已知布局伪造一个 reader。选择哈希落入相邻 bucket 的两个键后，可通过 `map.set()` 改写伪对象的 `m_indexed_elements`，先获得受限地址读写，再将目标 `ArrayBuffer` 的 data 指针改成任意地址，升级为完整任意读写。

有了任意读写后：

1. 从对象 vtable 泄露 LibJS 基址；
2. 读取 `free@GOT` 得到 libc 基址；
3. 读取 libc 的 `environ` 定位用户栈；
4. 在栈上布置 ROP，调用 `execve("/bin/sh", ...)`；
5. shell 命令从当前 JavaScript 的 `SEKAI_SOLVE_B64_BEGIN/END` 区间提取原生 ELF，写入 `/tmp/sekai_solve` 后执行。

### 2. QEMU TCG LPE：VM86 `iret` 权限绕过

这一段容易被误写成 Linux 内核漏洞，实际漏洞仍在 QEMU 的 x86 TCG 模拟器。`helper_ret_protected()` 处理 32 位 `iret` 时，只要攻击者提供的 `EFLAGS` 带有 `VM` 位，就会过早跳到 `return_to_vm86`；该路径尚未拒绝从 CPL3 用户态进入 VM86，却允许从伪造的 `EFLAGS` 同时载入 `IOPL_MASK`。

利用流程如下：

1. 从 64 位用户态以 `lretq` 切到 32 位兼容模式；
2. 构造 `VM=1、IOPL=3` 的 `iretd` 栈帧，进入 VM86；
3. 在 16 位代码中通过 `sysenter` 进入 Linux 兼容系统调用路径，再借修改后的 vDSO 落点回到 32 位；
4. 以远返回切回 64 位，此时 `IOPL=3` 仍保留；
5. 利用端口 I/O 和 TCG 映射技巧取得客体物理内存读写，定位并修改客体内核 `setuid` 检查分支，随后调用 `setuid(0)`、`setgid(0)` 获得 root。

root 只是为了能够直接枚举 PCI、配置 virtqueue 并操作 `virtio-snd`；客体内没有目标 flag。

### 3. QEMU `virtio-snd` 逃逸

`qemu_1.patch` 从 `virtio_snd_pcm_in_cb()` 移除了实际缓冲区上限：

```cpp
// 原本还会限制 max_size - buffer->size
size = audio_be_read(stream->s->audio_be,
                     stream->voice.in,
                     buffer->data + buffer->size,
                     MIN(available,
                         stream->params.period_bytes - buffer->size));
```

`period_bytes` 可由客体设置，因此 host audio backend 可以越过 `VirtIOSoundPCMBuffer::data` 写入后继宿主堆对象。题目没有提供 OtterSec 原利用使用的 `virtio-9p`，官方解改为攻击 TCG 的 `CPUTLBEntry` 软件 TLB 表。

关键布局是：

- 最小 user-mode TLB 有 64 个、每个 `0x20` 字节，总请求 `0x800`，对应 glibc `0x810` chunk；
- 源 RX 缓冲区令 `in_len=0x3d8`，其分配落入 `0x410` chunk；
- 设置 `period_bytes=0x3f7` 后，audio 写入从偏移 `0x29` 开始，恰好越过源 chunk `0x10` 字节；
- 这 16 字节正好覆盖后继 `CPUTLBEntry[0]` 的 `addr_read` 与 `addr_write`，而 `addend` 保持不变。

先让 guest 虚拟地址 `0x8000000` 在 64 项表中占据索引 0，再把两个比较字段清零后，访问 guest NULL 页也会命中该条目，计算出的 host 地址却仍使用原来的 `addend`，从而把 QEMU 堆页映射为客体的“窗口”。

为了让目标对象相邻，原生载荷使用不同 PCM stream 分别控制释放时机：

1. 用 TX buffer 喷射 `0x810` 与 `0x410` chunk，并先把 user-mode TLB 扩大；
2. 排列 `[0x410 源 RX][0x810 目标 RX]`，只释放目标；
3. 控制 TLB 的 100 ms 统计窗口和 large-page flush，使表缩到 64 项并回收该 `0x810` 空洞；
4. 启动源 stream，让越界的 16 字节清零 `CPUTLBEntry[0]`；
5. 从 NULL 窗口泄露 QEMU text 与 TCG RWX code-cache 地址，并找到同页的 `tcache_perthread_struct`；
6. 篡改 `entries[0x3f]` 后，以 `0x410` TX 分配完成定向宿主写；
7. 向 TCG RWX 区写入调用 `system` 的 stub，将 `helper_info_fninit.func` 指向该 stub；
8. 客体执行 `FNINIT` 触发回调，在宿主运行 `/readflag sekai 3in1`。

作者的[完整题解](https://qyn.app/blog/sekaictf-2026-3in1/)还给出了 VM86 状态切换和 TLB 缩容的源码级推导；上述关键漏洞、对象尺寸和最终控制流已经完整归纳在正文中。

整个链条可表示为：

```text
JavaScript
  -> Set.intersection UAF
  -> LibJS arbitrary read/write + ROP
  -> guest native payload
  -> QEMU VM86 bug -> IOPL=3 -> guest physical R/W -> guest root
  -> virtio-snd 16-byte host heap overflow
  -> corrupt TCG software TLB
  -> host heap window + tcache targeted write
  -> helper_info_fninit -> RWX stub -> system
  -> /readflag sekai 3in1
```

最终得到：

```text
SEKAI{w0w_y0u_35c4p3d_libjs_linux_4nd_qemu_b3tter_th4n_https://osec.io/blog/2026-03-17-virtio-snd-qemu-hypervisor-escape/}
```

## 方法总结

“3in1”不是三个互不相关的漏洞，而是一条权限逐层扩大的链。分析时必须区分漏洞所在组件与漏洞产生的权限结果：第一段是 LibJS UAF；第二段是 QEMU CPU 模拟错误，却用于提升客体权限；第三段才是 QEMU 设备模型的宿主堆越界与逃逸。

第三段最值得复用的思路是，在缺少额外设备对象时不必强求传统任意读写对象。TCG 软件 TLB 本身决定“guest 地址映射到哪个 host 地址”，只需精确清零比较字段并保留 `addend`，就能把很弱的、内容近乎不可控的越界转化为宿主内存窗口，再利用分配器元数据升级为定向写。
