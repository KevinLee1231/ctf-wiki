# GlacierCTF 2025 notally_safe

## 题目简述

题目是一个 C 编写的笔记管理器，可以把 SQLite 数据库导入内存中的 `Note` 数组。程序要求数据库 schema 中标题为 `VARCHAR(128)`、正文为 `VARCHAR(336)`，还执行完整性检查，因此看起来导入长度受控。

但 SQLite 的 `VARCHAR(n)` 不会强制长度。导入函数以 `sqlite3_column_bytes()` 为长度，直接 `memcpy` 到固定数组，没有与 128、336 比较。攻击者可提交结构完全合法、内容超长的 SQLite 数据库，获得稳定的堆溢出，再通过堆布局、任意读和 tcache poisoning 将 ROP 写到栈上。

## 解题过程

### 1. 构造能通过检查的越界数据库

内存对象的核心布局为：

```c
struct Note {
    size_t id;
    char title[128];
    char content[336];
};
```

导入器验证的是 `sqlite_master` 中的建表字符串和 `PRAGMA integrity_check`，但复制逻辑近似：

```c
memcpy(note->title,
       sqlite3_column_text(stmt, 1),
       sqlite3_column_bytes(stmt, 1));
memcpy(note->content,
       sqlite3_column_text(stmt, 2),
       sqlite3_column_bytes(stmt, 2));
```

因此 exploit 用正常 SQLite API 创建与题目逐字相同的表，再插入超长 TEXT/BLOB。数据库在 SQLite 看来没有损坏，读取到 C 结构时才越界，这也是绕过点。

### 2. 第一阶段泄漏 heap 并修复被破坏的 chunk

程序的 NoteList 通过 `zrealloc` 进行 `malloc → copy → free`，SQLite 自身也会产生大量临时分配。参考 solver 先用多轮导入稳定这些噪声，再让一个 Note 邻接可进入 unsorted/tcache 的约 `0x400` chunk。

首个超长正文覆盖相邻 chunk 的 size，同时保留 glibc safe-linking 需要的伪指针形态。tcache 中“编码后的 NULL”实际是：

$$
\operatorname{PROTECT\_PTR}(pos,0)=pos\gg12.
$$

将该值通过笔记显示功能泄漏后左移 12 位，可恢复对应堆页地址。参考 exploit 随后把被碰坏的 size 修回 `0x401`，避免 allocator 在下一次操作时提前终止。

### 3. 伪造 NoteList 获得任意读

在可控的约 `0x400` 区域中布置 `0x1e0`、`0x210` 等伪 chunk，并在后续导入中重放程序期望的相邻堆指针，使某个 Note 与 NoteList 的 backing store 发生重叠。伪造列表元素指针后，列表展示路径会把攻击者指定地址解释为 `Note *` 并读取其 `id`，形成 8 字节任意读。

利用该原语依次读取：

1. unsorted-bin/main_arena 指针，按随附件 libc 的偏移 `0x203bf0` 计算 libc base；
2. libc 中 `__libc_argv`（参考偏移 `0x2046e0`），取得当前进程栈地址；
3. 栈附近数据，定位菜单调用链的返回地址。

这些偏移只适用于题目提供的 libc；换环境时应从对应 ELF 符号和实际泄漏重新计算。

### 4. tcache poisoning 将 Note 分配到返回地址

恢复列表结构并释放用于布局的 Note 后，选择合适 size class。safe-linking 下，写入空闲 chunk 的 next 不能直接填栈地址，而应编码为：

$$
next_{enc}=target\oplus(chunk\_addr\gg12).
$$

通过另一次 SQLite 正文溢出覆盖该 next。随后两次同尺寸分配：第一次取走原 chunk，第二次让 Note 的数据区落到栈上返回地址附近。最终导入仍走那条无界 `memcpy`，把对齐用的 `ret`、设置参数的 libc gadget、`system` 和 `/bin/sh` 指针写成 ROP 链。菜单函数返回后获得 shell并读取 flag：

```text
gctf{m4554g1ng_h3ap_chunks_w/_VARCHAR_OOB_h45_n0t4bl3_h3alth_b3n3f1t5}
```

## 方法总结

本题最重要的误区是把 SQL 类型声明当作 C 缓冲区边界。SQLite 的类型亲和性和 schema 完整性都不能代替显式长度检查。完整利用依次建立“合法数据库触发 OOB → heap 泄漏 → 伪列表任意读 → libc/stack 泄漏 → safe-linking tcache poisoning → 栈上 ROP”，每个阶段都在进入下一阶段前修复 allocator 关键元数据。正确修复是在每次复制前验证 `column_bytes`，超长即拒绝，或使用按实际长度分配的内存对象。
