# mirage

## 题目简述

服务以 Selenium 打开攻击者提供的 HTTP(S) 页面，使用固定 Chromium commit `3c0517dbb233391c728684395d9aebb099665a09`。目标是突破 V8 renderer 的 heap sandbox，重新启用默认关闭的 Mojo JavaScript binding，再调用专门的 Flag Mojo 接口。官方 WP 将根因定位为 Turboshaft 的 stack argument load elimination：一个栈槽先按 `WordPtr` 加载、再转换为 tagged value 时，优化器忽略了两次读取之间 GC 可能移动对象，留下指向已释放 V8 heap 对象的句柄。

这是一条浏览器 renderer 内存破坏链，核心能力是 heap 任意读写及 sandbox escape，归入 Pwn 而非普通 Web。

## 解题过程

### 从优化错误获得 V8 heap 读写

官方 HTML 先喷射 `ArrayBuffer` 并多次 GC，使对象进入 old space，同时准备浮点数组 map。函数 `f(...args)` 被反复调用以触发 Turboshaft 优化；在两个相关栈参数读取之间制造浮点数组 spray。发生移动后，`t[0][0] !== t[0][1]` 表示旧指针与新 tagged 值错配，得到可伪造的浮点数组视图。

将伪造数组的 map、length、elements store 指向 sprayed old space 后，脚本扫描并改变一个 ArrayBuffer 的 `byteLength`/元素描述，从有限 OOB 读写扩大到被污染 ArrayBuffer 的读写。随后把对象数组与浮点数组复用同一 element backing store，实现 `addrof`。

### 逃出 heap sandbox 并读取 flag

题目 patch 让 ArrayBuffer/DataView 回到不使用 bounded size 的行为。利用已得地址，官方脚本把 DataView 的 offset 设为 ArrayBuffer backing-store offset 的相反数，取得越过 heap sandbox 的任意读写。它调用题目暴露的 `leak()` 得到 isolate 地址，读取 Chrome 指针并将 `is_mojo_js_enabled_` 标志写为非零。

页面刷新后 `Mojo` 可用，脚本绑定 `blink.mojom.Flag` 接口并调用 `getFlag()`，再以对自身页面的请求带回结果。具体 V8 map、isolate 和 Chrome 偏移均硬编码于官方 HTML，绑定该 Chromium 构建，正文不将其误写为通用偏移。

### 验证

官方解法预期回收 `DUCTF{w0w_th15_w0rk5_1n_r3l3453!}`。本文未部署网页、触发 JIT 或访问远端 bot；漏洞成因与利用链由官方 `WRITEUP.md`、`solution.html` 和本地 runner 静态复核。

## 方法总结

- 核心技巧：JIT load elimination 不能跨越会移动对象的 GC 假设；WordPtr 与 tagged 转换混用时，旧地址可成为 UAF 入口。
- 识别信号：V8 专题给出特定 commit、GC spray、ArrayBuffer/DataView 与 Mojo 接口时，应将 JIT primitive、sandbox escape 和高权限接口启用分段分析。
- 复用要点：map、old-space 和 Chrome 内部字段偏移是构建特异的；先确认 OOB 视图、再做 addrof、再扩展到 sandbox 外读写，避免把后续接口调用误当成漏洞本体。
