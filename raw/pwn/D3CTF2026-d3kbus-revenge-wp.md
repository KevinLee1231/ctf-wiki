# d3kbus-revenge

## 题目简述

`d3kbus-revenge` 保留了 `d3kbus.ko` 中预期的 CRC32C 回写漏洞，但修复了普通版可以绕过模块、直接利用 QEMU TCG 越界读写提权的非预期路径。因此，解题重点仍是把零拷贝消息总线中的 deferred CRC trailer 落到只读文件的 page cache，再将受控 CRC 转换为 4 字节覆盖。

模块会检查 external segment 对应的 inode、mapping、page index 和 snapshot 是否仍然一致，却没有验证来源文件是否可写。攻击者可通过只读 fd 把 BusyBox 页面送入消息总线，令内核以“提交 CRC”的名义修改其 page cache。

## 解题过程

### 1. 构造可回写的 external segment

利用需要同时满足三个条件：

1. 使用 `sendfile(producer_fd, file_fd, ...)`，让内核内部 splice pipe 保留来源 page-cache page；显式用户管道会触发私有复制。
2. 创建两个启用 shared external 与 CRC 的订阅者，使 deferred trailer 继续引用 external segment。
3. 输入 payload 长度设为 24 字节，投影窗口只取前 20 字节；窗口覆盖完整 segment 时同样会被私有化。

关键参数为：

```text
channel.max_subscribers = 2
subscriber[0].flags      = 0x02 | 0x04
subscriber[1].flags      = 0x02 | 0x04
window_offset            = 0
window_length            = 20
wire.payload_length      = 24
```

投影窗口前 16 字节参与 CRC32C，最后 4 字节作为 deferred trailer 写回原页面。若目标文件偏移为 $T$，则从 $T-16$ 开始送入 24 字节，CRC 正好覆盖 $[T,T+4)$。

### 2. 反解 CRC32C 得到任意 32 位写

projected header 中的 `user_tag` 完全可控。固定其余字段和 16 字节 payload prefix 后，CRC32C 是从 32 位 `user_tag` 到 32 位输出的仿射映射：

```text
F(t) = CRC32C(projected_header(user_tag=t) || payload_prefix)
base = F(0)
delta_i = F(1 << i) xor base
```

对于目标值 `wanted`，在 GF(2) 上求解：

```text
Σ(t_i · delta_i) = wanted xor base
```

本题参数下映射满秩，因此每个 32 位目标值都有唯一的 `user_tag`。每次覆盖前重新计算 CRC，即可确认写回值与目标一致。

### 3. 改写 BusyBox 的 root 执行入口

BusyBox 是固定地址的静态 ET_EXEC。低权限 shell 退出后，root 启动脚本会执行：

```sh
poweroff -d 0 -f
```

因此可从 `poweroff_main` 附近的 4 字节对齐位置开始，连续执行 16 次 CRC 覆盖，写入约 64 字节的用户态 shellcode。shellcode 完成：

```c
fd = open("/flag", O_RDONLY);
n = read(fd, rsp - 0x80, 0x7f);
write(STDOUT_FILENO, rsp - 0x80, n);
exit(0);
```

覆盖完成后结束 `ctf` shell，root 启动流程继续调用 `poweroff`，被污染的 BusyBox 页面随即执行上述代码并输出 flag。该链只修改用户态静态文件的 page cache，不依赖内核地址泄漏、ROP、SMEP/SMAP 绕过或关闭 KASLR。

### 4. revenge 修复的 QEMU 路径

普通版使用不带 KVM 的 QEMU TCG。旧版 `access_ptr()` 先执行 `size - len`：

```c
if (offset <= ac->size1 - len)
    return ac->haddr1 + offset;

assert(offset <= ac->size - len);
```

当 x87 的 10 字节 `FLD/FSTP` 跨越页尾时，`len` 可能大于剩余范围，减法发生无符号下溢，使边界判断错误放行。普通版可由此取得宿主越界读写，再修改来宾页表和内核权限检查。

revenge 将判断改为先验证 `offset`，再做不会下溢的减法：

```c
if (offset <= ac->size1 && len <= ac->size1 - offset)
    return ac->haddr1 + offset;

assert(offset <= ac->size && len <= ac->size - offset);
```

跨入第二页时还必须确认第二个 host buffer 存在。这样切断了 QEMU 平台层非预期，但没有改变 `d3kbus` 的 page-cache CRC 回写逻辑，预期利用链仍然成立。

## 方法总结

- 核心技巧：利用消息总线把 CRC 完整性提交错误地作用于只读 external page，并通过 GF(2) 线性代数反解 `user_tag`，将校验值写回提升为任意 32 位覆盖。
- 识别信号：零拷贝 external page、snapshot 一致性检查、deferred checksum trailer 与只读文件 fd 同时出现时，应检查“页面仍然相同”是否被错误地当成“允许修改页面”。
- revenge 差异：修复点位于 QEMU TCG 的跨页访问边界判断，目标是移除虚拟化平台 0day；模块的预期漏洞和 BusyBox `poweroff_main` 触发方式不变。
