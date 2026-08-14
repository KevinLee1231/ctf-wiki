# bi0sCTF 2024 - palindromatic

## 题目简述

题目是一个 Linux 内核 palindrome 检查模块。每个 0x400 字节 request 含 `type`、随机 `magic`、原字符串和净化字符串，并可在 incoming/outgoing 两个环形队列之间移动。`SANITIZE` 使用 `strcpy` 写满 `sanstr` 时会在对象末尾再写一个零字节，覆盖相邻 slab 对象的 `type` 低字节。

损坏后的 request 在 `PROCESS` 中走到异常控制流，同时进入两个队列，最终形成 UAF/重复释放原语。官方利用通过 cross-cache 将 0x400 request 与 `pipe_buffer`、`msg_msgseg` 复用，泄漏并伪造 `pipe_buffer`，以 Dirty Pipe 风格覆盖 `/etc/passwd` 获得 root。

## 解题过程

### 从单字节溢出制造双队列引用

`request_t` 恰好来自专用 0x400 slab。若输入的 `STRING_SZ` 个字节全是字母，净化结果也达到最大长度，以下调用会把结尾 NUL 写到对象边界之外：

```c
temp_buffer[ptr] = 0;
strcpy(req->sanstr, temp_buffer);
```

连续 spray request 后，越界零落入下一对象的 `type`。正常类型从 `RAW = 0x1337` 开始；低字节被清零后，它既不等于 `RAW`，也不等于 `SANITIZED`。`pm_process_request()` 先无条件把指针加入 outgoing queue，但只有两个合法类型分支才会从 incoming queue 出队。因此损坏对象同时存在于两个队列。

官方利用先记录队列容量，反复 `PROCESS`，直到容量变化不符合正常规律，以此定位损坏对象。随后 `RESET` 把它移到 incoming 队首，处理并 `REAP` 其余对象，留下一个指向已释放 request 的队列引用。

### 将 UAF 转成 `pipe_buffer` 重叠

大量创建 pipe 并写入少量数据，让内核分配 pipe ring。通过对悬空 request 连续 `RESET` 触发释放，再 spray 新 pipe，目标 0x400 slab 页会经 cross-cache 复用为 `pipe_buffer` 数组。读取各 pipe 的首字节，若原本写入 `AA` 的 pipe 读出 `B`，即可识别被 UAF 影响的 victim。

关闭其余 pipe 后保留 victim 的悬空引用，再 spray 大尺寸 System V 消息，使 `msg_msgseg` 占据释放的 `pipe_buffer`。对只读打开的 `/etc/passwd` 执行 `splice`，内核会更新 victim pipe 中的真实 `pipe_buffer`；随后 `msgrcv` 读回重叠消息，便可取得完整结构，包括 `page` 和 `ops` 指针。

### 伪造可合并 pipe 并覆盖文件

复制泄漏出的结构，保留其合法页面与操作表指针，只修改：

```c
fake_pipe.flags = PIPE_BUF_FLAG_CAN_MERGE;  /* 0x10 */
fake_pipe.offset = 0;
fake_pipe.len = 0;
```

再用 `msgsnd` 把伪结构 spray 回重叠位置。此时向 victim pipe 写数据会被合并到它引用的文件页，而不是分配新匿名页。将一条带已知密码哈希的 root 记录写到 `/etc/passwd` 对应页，然后使用该密码切换 root，即可读取 flag。

利用过程中的关键验证点是：

- `QUERY` 确认确实存在双队列 request；
- pipe 内容差异确认 victim，而不是假设固定编号；
- `msg_msgseg` 泄漏位置不再全为 spray 填充值；
- 伪造前保留原始 `page`、`ops` 和其他必要字段，只改变合并相关字段。

## 方法总结

一个对象末尾的 NUL 溢出足以破坏相邻 request 的枚举类型。异常类型让同一指针同时进入两个队列，队列操作再把它转成可重复利用的 UAF。由于 request 来自 0x400 专用 cache，利用需要 cross-cache 把页面迁移给 pipe 与消息对象；最终借可合并 `pipe_buffer` 覆盖文件页。每个 spray 阶段都应通过容量、内容或泄漏特征动态定位 victim。
