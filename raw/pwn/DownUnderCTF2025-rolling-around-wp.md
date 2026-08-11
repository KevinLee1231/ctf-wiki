# Rolling Around

## 题目简述

题目提供基于指定 Linux commit 构建的 QEMU 内核、启动脚本和内核补丁。补丁新增 eBPF `ROL` 指令，并同时为 verifier 编写寄存器范围更新逻辑；QEMU 启用 SMEP、SMAP，内核地址随机化需要通过信息泄漏处理。

漏洞不在真实的 rotate-left 运行语义，而在 verifier 对范围的抽象：它把寄存器的最小值和最大值分别旋转，仿佛 rotation 保持普通数值区间。位旋转并不单调，这会使 verifier 认可的范围和运行时实际值脱节。补丁还注释掉 `sanitize_ptr_alu()` 中的投机路径检查；这是题目为使利用可行而刻意放宽的保护，不能将该利用结论外推到未修改的上游内核。

## 解题过程

### 制造 verifier/runtime 范围混淆

官方 exploit 从 map 读取一个运行时为 0、但 verifier 只能视为未知的值；加 4、与 7 相与，再以条件跳转收窄 verifier 的范围。运行时值仍是 4。对该寄存器执行 `ROL 62` 后，实际值会变为低位的 1，而 verifier 却根据分别旋转的上下界给出不相容的高位范围。后续加法和掩码让 verifier 认为偏移为安全的小值，实际却得到可越界的值。

简化后的关键指令序列是：

```c
v = map_value[1];     // runtime 为 0，verifier 视为未知
v += 4;
v &= 7;
if (v < 2) return 0;  // 收窄抽象范围，runtime 仍为 4
v = rol64(v, 62);     // verifier 的 min/max 更新错误
/* 后续运算使实际 offset 与 verifier 认知脱节 */
```

这构成针对 map value 指针的越界读写 primitive；关键是访问要让 verifier 将该偏移看作非恒定且在界内，避免被常量重写或边界钳制。

### 绕过 KASLR 并建立任意读写

首先将越界 map 指针向 value 区之前移动，读取 `bpf_map->ops` 中的 `array_map_ops` 函数表指针，得到内核文本地址。接着把 `bpf_map->btf` 覆写为目标地址减去 `btf.id` 偏移，调用 `BPF_OBJ_GET_INFO_BY_FD`；内核会把伪造 `btf` 的 `id` 返回到用户态，因而形成按 32 位读取的任意读。

官方代码不依赖固定 `init_pid_ns` 地址：它扫描 kernel string/symbol table 找到 `init_pid_ns`，再按当前 PID 在命名空间 IDR 中寻找对应 `task_struct` 并读取 `cred` 指针。这一步把 KASLR 下的 map-ops 泄漏转化为当前进程凭据的准确地址。

任意写通过第二次 map 元数据破坏取得。利用将目标 map 的 `ops` 指向攻击者可控的 map value，伪造函数表；再把 `map_type` 改为 stack，并将 stack 的 `push_elem` 槽替换成 array map 的 `get_next_key`。调用 `BPF_MAP_PUSH_ELEM` 时，攻击者控制的 `key` 和 `next_key` 被错误解释，得到任意 32 位内核写。最终将当前 `cred` 中的 uid、gid、euid 写为 0。

### 验证

官方 `exploit.c` 依次打印 map 指针、`array_map_ops` 泄漏、任意读测试、`init_pid_ns` 与当前 `cred` 地址；随后执行 `overwrite_cred()`，清理被临时破坏的 map 字段并启动 `sh`。这些输出和 root shell 是验证链。本文只审阅补丁、官方说明与 exploit 源码，没有编译或启动内核。

## 方法总结

- 核心技巧：攻击 verifier 的不精确抽象，而不是 eBPF 指令解释器本身；非单调位操作绝不能靠独立旋转 min/max 建模。
- 识别信号：内核补丁新增 ALU 指令，同时修改 `scalar_min_max_*`、tnum 或 pointer sanitation 时，应先比较 verifier 状态和实际执行值是否可分离。
- 复用要点：eBPF 内核利用通常按“范围混淆 → map/文本泄漏 → 任意读 → 元数据 type confusion 任意写 → cred 修改”逐层构造，并且要明确标注题目额外削弱的防护，避免错误泛化。
