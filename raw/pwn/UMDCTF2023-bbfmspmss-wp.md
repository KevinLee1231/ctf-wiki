# Bill's Blazingly Fast Memory Safe Pocket Monster Storage System

## 题目简述

题目是一个用 Rust 编写的宝可梦存储系统，支持创建、删除箱子以及按槽位存取记录。虽然源码没有使用 `unsafe`，但箱子名未经限制地参与路径拼接，使程序能够打开预期目录之外的任意可读写文件；结合 `/proc/self/mem` 可以把合法文件操作转化为进程内存写入。

## 解题过程

`box_path()` 先创建 `./boxes/`，随后直接执行：

```rust
path.push(name);
```

当 `name` 是绝对路径 `/proc/self/mem` 时，`PathBuf::push` 会用它替换原路径。`deposit()` 随后以读写方式打开该文件，并把用户给出的槽位乘以固定记录大小：

```rust
const MAX_POKEMONSTER_SIZE: u64 = 32;
let offset = slot.checked_mul(MAX_POKEMONSTER_SIZE)?;
file.seek(SeekFrom::Start(offset));
bincode::serialize_into(&file, &new_poke);
```

因此槽位决定写入地址的 $32$ 字节对齐部分，而序列化结构开头的 `number: usize` 提供可控的前 $8$ 字节。即使目标映射本身不可写，通过 `/proc/self/mem` 写入仍可修改当前进程地址空间。

程序启用了 PIE，官方利用脚本先把 `/proc/self/mem` 当作探测器：在典型 PIE 区间
`0x550000000000` 到 `0x570000000000` 中尝试写入，以是否出现
`Pocket Monster deposited!` 判断地址是否落在有效映射。脚本依次用大批量、单步映射跨度和页粒度搜索，最终得到精确 ELF 基址。

得到基址后，利用脚本选择删除箱子时必然调用的 `Vec::retain`，其版本对应偏移为
`0xd090`。由于写入地址必须 $32$ 字节对齐，实际从函数内的
`base + 0xd0a0` 开始覆盖。`/bin/sh` shellcode 被改写为若干相隔 $32$ 字节的短块，每块前 $8$ 字节作为 `number` 写入，中间用跳转跨过不可控区域。

最后删除任意现有箱子以触发：

```rust
state.boxes.retain(|b| b.name != name);
```

控制流进入覆盖后的代码，执行 `execve("/bin/sh", 0, 0)`。在 shell 中读取
`flag.txt` 得到：

```text
UMDCTF{b1ll_forg0r_ab0ut_pr0c_s3lf_m3m_sku11}
```

## 方法总结

- 核心漏洞不是 Rust 内存安全缺陷，而是未约束绝对路径导致 `/proc/self/mem` 被当作普通箱子文件打开。
- `/proc/self/mem` 的读写结果还可充当无回显地址探针，逐级缩小 PIE 基址范围。
- 当写入原语受对齐和长度限制时，可把 shellcode 分块，并用短跳转跨越不可控间隙。
- 依赖固定函数偏移的利用只适用于题目给定构建；更换编译器或优化参数后必须重新确定目标偏移。
