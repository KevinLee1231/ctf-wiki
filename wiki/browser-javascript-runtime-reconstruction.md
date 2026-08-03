---
type: technique
tags: [reverse, web, javascript, browser, runtime, deobfuscation]
skills: [ctf-reverse, ctf-web]
raw:
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
  - ../raw/web/game-state-websocket-and-wasm.md
updated: 2026-08-03
---

# Browser JavaScript Runtime Reconstruction

## 适用场景

关键校验、解密、签名或 VM 逻辑位于浏览器 JavaScript，且依赖 DOM、storage、cookie、WebCrypto、worker、WASM、网络响应或浏览器环境。目标是重建能重现关键状态的最小运行边界，而不是一开始就美化整个 bundle。

## 识别信号

- 压缩 IIFE、大型常量表、字符串索引器、dispatcher 或自定义 bytecode loop。
- 同一函数在 Node 中与浏览器中结果不同，或缺少 `window`、`document`、`navigator`、`crypto.subtle`、`localStorage` 后立即失败。
- 关键值由 `fetch`/XHR/WebSocket、worker message、WASM memory 或页面初始状态注入。
- 静态搜索能看到调用点，但参数只在运行时生成。

## 最小证据

- 保留发起请求的 script/initiator、请求和响应、cookie/storage 初值以及能触发目标逻辑的最小 UI 动作。
- 在稳定边界捕获输入、输出和必要环境值；例如签名函数参数、解密前后 buffer、VM PC/state 或 WASM memory 切片。
- 重建脚本与真实页面对同一输入产生一致输出，或能指出首个不一致的状态。

## 解法骨架

1. **Observe**：先在 DevTools 中记录真实请求、initiator、脚本加载顺序、worker/WASM 以及触发动作。
2. **Capture**：在命名函数、Web API、网络边界或稳定的数据结构上下断；只保留重建需要的参数、buffer 和环境快照。
3. **Rebuild**：把目标函数和必要常量放入最小 Node/浏览器 harness；对 DOM、storage、network、time/random 和 WebCrypto 依赖分别建模。
4. **Patch**：每次只补一个环境依赖，并记录第一个分歧点；不要用一个巨大的假 `window` 把错误吞掉。
5. **DeepDive**：只在最小重现已稳定后，再用 AST rename、constant folding、dead-code elimination 或 VM lifting 扩展理解。
6. 用原页面输出、已知请求或 forward checker 做最终验证，并保留必要的 network/storage 前置条件。

## 边界表

| 主要障碍 | 归属 |
|---|---|
| 压缩/混淆脚本、JS VM、WASM host bridge 的行为还原 | `ctf-reverse` |
| 同源、CSP、DOM XSS、admin bot、认证或 HTTP 业务逻辑 | `ctf-web` |
| 脚本的恶意投递、窃密、C2、持久化或配置恢复 | `ctf-malware` |

## 常见陷阱

- 先做全量 beautify/rename，却没有保留原始 bundle、source map、network 和触发状态。
- 把 `Date.now`、random、locale、Unicode、TypedArray 端序、WebCrypto 或 storage 差异误判为算法错误。
- 在 Node 中用过度 mock 让脚本“不报错”，却没有证明语义一致。
- 页面中存在诱饵函数，却没有用 initiator、xref 或实际调用 trace 确认关键路径。

## 关联技巧

- [custom-vm-and-wasm-state-lifting.md](custom-vm-and-wasm-state-lifting.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)
- [xss-dom-and-browser-tricks.md](xss-dom-and-browser-tricks.md)
- [trace-hook-and-state-snapshot-reconstruction.md](trace-hook-and-state-snapshot-reconstruction.md)

## 原始资料

- [android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
- [game-state-websocket-and-wasm.md](../raw/web/game-state-websocket-and-wasm.md)
