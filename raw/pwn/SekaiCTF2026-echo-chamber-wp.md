# Echo Chamber

## 题目简述

目标是一个自制的多线程 HTTP/1.1 服务。主线程负责 `accept()`，每个新连接交给一个持久 worker；带 `Connection: keep-alive` 的连接会在同一 worker 中继续处理请求。运行脚本设置：

```text
glibc.malloc.tcache_count=0
glibc.malloc.tcache_max=0
glibc.malloc.arena_max=8
```

因此利用不仅要处理负 `Content-Length` 引发的堆溢出，还要稳定控制请求落入哪个 worker 和 glibc arena，并在没有 tcache 的条件下构造写原语。

## 解题过程

### 1. 根因：`-1` 同时造成最小分配、超长读和超长回显

POST 处理逻辑为：

```c
size_t body_len = strtoll(content_length->val, &end, 10);
body = malloc(body_len + 1);
size_t nbytes = readn(self->fd, body, body_len);
body[nbytes] = 0;
handle_post(self->fd, path, headers, body, body_len);
```

当 `Content-Length: -1` 时：

- `strtoll()` 产生 `-1`，赋给 `size_t` 后变成 `SIZE_MAX`；
- `body_len + 1` 无符号回绕为 0，`malloc(0)` 只返回一个最小 chunk；
- `readn()` 的长度参数是 `unsigned int`，`SIZE_MAX` 又被截成 `UINT_MAX`，攻击者可向最小 chunk 后持续写入；
- `handle_post()` 最后把同一 `body` 按超长长度回显，因此损坏后的相邻堆内容还能从响应体泄露出来。

所以这不是“解析器接受负数”这么简单，而是一处由三次整数转换共同形成的堆任意长顺序覆写与越界回显。

### 2. 固定 worker 与 arena

worker 会被复用，而 glibc 最多建立 8 个 arena。官方脚本先创建多轮 keep-alive 连接并发送正常请求，将每个连接记录为 `arena_slots[0..7]`。后续所有“负长度溢出、阻塞中的 body、释放某块、触发某块”操作都从指定 slot 取连接，避免线程调度改变堆布局。

这里还利用了请求可被故意阻塞的特性：先发送正文的全部内容但保留最后一个字节，使 worker 已经完成 header/body 分配，却尚未进入回显和 `free()`；在另一条连接完成堆破坏后，再补上最后一个字节，便能按需要触发分配、释放或回显。

### 3. unsorted-bin 泄露

泄露阶段在目标 arena 中排列两个 `0x510` chunk `P`、`R` 及其 `Header` 对象。负长度 body 从前方最小 chunk 溢出，修复必要的 Header 字段，同时把 `P` 至 `R` 的区域伪造成一个较大的合法 free chunk。随后：

1. 触发释放，使伪造块进入 unsorted bin；
2. 再分配一块将其切割，让 remainder 与仍在使用的 `R` body 重叠；
3. 完成 `R` 的请求，服务原样回显 `R`，前 16 字节即 unsorted `fd/bk`。

在 main arena 中，链表指针可计算 `main_arena` 与 libc 基址，额外的堆指针用于确定主堆位置；在 non-main arena 中，同样的重叠直接泄露该 arena 及其 `heap_info` 基址。

### 4. 从 largebin top pivot 到受约束任意地址写

官方解没有假设可以一次覆盖任意 libc 地址，而是在每个 non-main arena 内建立可重复的“remainder header 写”：

1. 释放一个带 guard 的 `0x710` 大块，并用 `0x810` 分配把它整理进 largebin；
2. 负长度溢出同时伪造已排序 largebin chunk 和当前 top chunk；
3. 设置 `bk_nextsize`，利用 largebin unlink 的副作用把 `arena->top` 改到已知堆地址；
4. 通过一个预先阻塞的 body 重写 non-main `malloc_state`，包括 `top`、bins、`system_mem`；
5. 将伪造 top 的大小改成“到目标地址前 8 字节的距离 + 预期 remainder 大小”；
6. 触发下一次 top 分割，glibc 会在 `remainder + 8` 写入新 chunk 的 `size` 字段。

由此得到 `*write_addr = write_value`。限制是 `write_value` 必须看起来像合法 chunk size，即 16 字节对齐并带 `PREV_INUSE`；官方脚本通过挑选 pointer guard，让后续所有被 mangling 的函数指针都满足这个“size-shaped”约束。

### 5. TLS、取消路径与退出处理器

glibc x86-64 的函数指针编码为：

$$
\operatorname{mangle}(p)=\operatorname{ROL}_{64}(p\oplus guard,17)
$$

脚本先根据目标 `exit` 反求一个满足写入约束的 `guard`，再计算同一 guard 下的 mangled `system`。随后分别使用新的 non-main arena 完成六组写入：

1. 主线程 TLS 的 `pointer_guard`，同时在该 arena 的堆中保存命令字符串；
2. libc 现有 exit handler 的函数字段，改成 mangled `system`；
3. 同一 exit handler 的参数字段，改成命令字符串地址；
4. forced-unwind 函数指针，改成 mangled `exit`；
5. 将 `global_libgcc_handle` 设为非空；
6. 修改主线程 TLS 的 `cancelhandling`，置上 canceled 状态。

下一次连接使主线程再次进入 `accept()` 这一取消点，触发 forced unwind；被改写的 unwind 函数先进入 `exit`，随后 exit handler 解码并调用 `system(command)`。远端命令遍历 `/proc/self/fd/*`，把 `id` 和 `/app/flag.txt` 输出到仍然存活的连接，解决“RCE 已发生但标准输出不一定对应当前 socket”的问题。

核心因果链为：

```text
Content-Length: -1
  -> malloc(0) + read(UINT_MAX) + echo(UINT_MAX)
  -> forged unsorted overlap
  -> libc / main heap / non-main arena leaks
  -> largebin unlink pivots arena->top
  -> top split produces size-shaped arbitrary write
  -> pointer_guard + unwind + exit handler
  -> system(command)
```

最终得到：

```text
SEKAI{the_best_time_to_write_challenges_is_always_during_the_ctf_:P_}
```

## 方法总结

本题的第一步应把每个整数转换逐项列清：`strtoll` 的有符号结果、`size_t` 回绕、`malloc` 参数回绕，以及传给 `readn/writen` 时截断到 32 位，四者缺一都无法准确说明漏洞。

关闭 tcache 并不会消除堆利用，只会把重点转向 arena、unsorted/largebin 和 top chunk。面对多线程网络目标，应先建立“连接—worker—arena”的可重复模型，再设计跨连接的阻塞与触发顺序。最终写原语即使只能写“像 size 的值”，也可以通过反求 pointer guard，让 mangled 控制流指针主动适配原语，而不必强求无约束任意写。
