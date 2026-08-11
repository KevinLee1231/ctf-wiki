# sign in

## 题目简述

账户系统以链表保存 `user_entry`。注册时分配 `user_t` 与 `user_entry_t`，却只初始化 `entry->user` 与 `entry->prev`，遗漏 `entry->next = NULL`。删除当前账户会先释放 user，再释放 entry；下一次按相同大小注册时 tcache 复用这两块 chunk，使新 entry 的未初始化 `next` 保留先前 user password 的八字节内容。攻击者可把它设置为非 PIE 主程序中合适的零值内存指针，伪造链表尾部的 uid 0 账户。

## 解题过程

### 制造 stale next 指针

官方 solver 的顺序是：注册用户名 `x`，密码写入 `p64(0x402eb8)`；以该账户登录并删除；再注册一次。由于 `free(user)` 后 `free(entry)` 的 tcache LIFO 顺序，新的 `user_t` 复用旧 entry，而新的 `user_entry_t` 复用旧 user；未初始化的 `next` 字段便承接旧 password，即 `0x402eb8`。

`0x402eb8` 由官方的只读搜索脚本在固定、无 PIE 的主程序地址范围中选出：该处内容可作为连续的零 QWORD 布局。它让 `sign_in` 遍历到等价于空 username、空 password、`uid == 0` 的伪造记录。

### 伪造 root 登录状态

最后调用登录操作，username 与 password 均发送八个零字节；`memcmp` 比较命中伪造记录，返回 uid 0。主循环只检查 `uid == 0` 就在菜单 4 调用 `system("/bin/sh")`，不再验证原始 root 的随机密码。

### 验证

官方 solver 的最终交互进入 shell；题目配置给出的 flag 为 `DUCTF{welcome_root!_9dbfa98e17b7af9dbc1}`。本文未运行二进制或远端服务，tcache 复用顺序、固定地址和登录路径均由源码及官方 solver 静态核对。

## 方法总结

- 核心技巧：链表节点遗漏初始化并与可控、相同大小的已释放对象复用时，旧字段可变成跨类型的结构指针。
- 识别信号：`malloc` 后只赋部分字段、free 顺序可控、随后同尺寸重分配以及遍历 `next` 的认证逻辑，应同时检查。
- 复用要点：此类 stale heap 内容依赖 allocator size class、free 顺序和 PIE 状态；不要把固定的 `0x402eb8` 当作通用 gadget 地址。
