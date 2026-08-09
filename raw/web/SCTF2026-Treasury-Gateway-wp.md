# Treasury Gateway

## 题目简述

外部 Gateway 不能直接访问只监听 `127.0.0.1:3005` 的 Vault 内部接口，但后台任务每秒请求一次 `/internal/compliance/export-snapshot`，并把包含 `reconciliation_token` 的 JSON 写入所有已归还的 512 字节 scratch buffer。外部可调用的 `/__route/audit` 使用 unsafe 零拷贝解析 `X-Route-Selector`，其中同时存在越界 slice 和跨 `await` 生命周期伪造。

攻击者用只有一个冒号的 selector 让 nonce 指针落到 header 逻辑范围之外，再用慢速 chunked body 保持请求挂起；buffer 已归还池中，被 refresh 写入内部报告，而 `DeferredTrace` 仍持有指向它的伪 `'static` 引用。调整 header 长度即可用 4 字节窗口逐字节扫描报告。

## 解题过程

### 1. 触发 nonce 越界 slice

预期 selector 格式为 `tenant:path:nonce`。若第二个冒号缺失，解析器令：

```text
second      = raw.len()
nonce_start = raw.len() + 1
nonce_len   = min(512 - nonce_start, 4)
```

随后使用 `from_raw_parts(raw.as_ptr().add(nonce_start), nonce_len)`。长度依据整个 scratch 容量 512 计算，而不是 `raw.len()`，所以 slice 虽越过 header 的 Rust 逻辑边界，仍指向同一 scratch 后部。审计响应会把这 4 字节编码为 `nonce_preview_hex`。

### 2. 利用错误生命周期让 refresh 覆盖 buffer

`defer_trace` 从 tenant pool checkout buffer、清零并复制 header，然后把 nonce slice `transmute` 成 `&'static [u8]`。tenant/path 被复制为 owned String，但 nonce 保留借用；函数却立即把 scratch checkin 回池。

调用方随后执行：

```rust
let trace = selector.defer_trace(header);
request.into_body().collect().await?;
let result = trace.finish();
```

用原生 socket 发送 `Transfer-Encoding: chunked`，只发 headers 而暂不发送终止块。服务端停在 `collect().await`，约 1.3 秒后后台 refresh 会把内部快照覆盖到池中该 buffer。此时再发 `0\r\n\r\n` 结束 body，`finish()` 解引用的已经是新内容。

### 3. 滑动 4 字节泄漏窗口

只有一个冒号时，$nonce\_start=header\_length+1$。使用固定 tenant/path 前缀并逐次多追加一个 `A`，窗口便每次向后移动一字节：

```text
pad=0 -> snapshot[offset     : offset+4]
pad=1 -> snapshot[offset+1   : offset+5]
pad=2 -> snapshot[offset+2   : offset+6]
```

为提高稳定性，可一批同时打开 16 个未结束的 chunked 请求，让同一次 refresh 覆盖整批 buffer，再统一结束请求。默认内部 JSON 长 451 字节，token 接近末尾；官方脚本从约 offset 51 开始扫描 430 个位置，不必硬编码 token 的精确偏移。

### 4. 重组连续快照

相邻 4 字节窗口有 3 字节重叠。对每个响应取第一个字节，最后再追加最后一个窗口的后三字节：

```python
stitched = b''.join(chunk[:1] for chunk in chunks)
stitched += chunks[-1][1:]
flag = re.search(rb'SCTF\{[^}]+\}', stitched).group()
```

这样重建出的连续区域包含 `reconciliation_token`。token 由部署环境动态提供，不应把源码默认值误写成比赛实例结果；以泄漏 JSON 中正则匹配到的值为准。

## 方法总结

本题需要两个缺陷同时成立：`from_raw_parts` 让 slice 越过 header，`transmute` 与提前 checkin 又让该 slice 在 buffer 复用后继续存活。若没有慢 body，refresh 通常来不及覆盖；若 buffer 尚未归还，refresh 也不会写入。Rust 的 unsafe 不会自动延长底层对象生命周期，异步代码中更不能把池化内存借用伪造成 `'static`。复现时应先用已知短窗口确认偏移滑动规律，再批量扫描，避免把调度抖动误认为 selector 长度计算错误。
