---
type: family
tags: [web, family, nodejs, prototype-pollution, vm-escape]
skills: [ctf-web]
raw:
  - ../raw/web/node-and-prototype.md
updated: 2026-06-12
---

# Node.js Prototype Pollution and VM Escape

## 作用边界

本页是 Node.js 原型污染、服务端 JavaScript gadget 和 VM/sandbox escape 的二级 family。共同点是：攻击者先污染对象结构或进入受限 JS 执行环境，再借框架、模板、库配置或模块系统把影响扩大到文件读、SSRF、模板注入或 RCE。

它不等同于普通 XSS，也不等同于所有 Node Web 题。只有当证据指向 prototype chain、对象 merge、Node VM、CommonJS/ESM loader、Happy-DOM 或服务端渲染 gadget 时才进入本页。

## 识别信号

- 请求体、query、JSON、YAML、表单或配置合并点能写入 `__proto__`、`constructor.prototype`、`prototype`。
- 依赖中出现 lodash、flatnest、merge/deepmerge、pug、ejs、vm2、Node `vm`、Happy-DOM、RSC 或 server-side JS sandbox。
- 污染后业务行为变化：模板选项、文件路径、命令参数、debug flag、AST 字段、require 路径或安全开关被影响。
- sandbox 内能访问 `constructor`、`process`、`Function`、`import()`、`module`、`require` 或错误栈。

## 最小证据

- 一个最小污染 payload 能让普通对象出现新属性，或让模板/库配置产生可观察差异。
- 能指出污染 sink：merge、deserialize、flatten/unflatten、模板渲染、DOM 仿真、VM 执行、RSC payload。
- 对 VM escape，要确认 Node 版本、模块类型 CommonJS/ESM、sandbox 暴露对象和禁用 API。
- 对 gadget，要能从污染字段走到具体危险行为，而不是只证明 `{}.polluted === true`。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| `__proto__` / `constructor.prototype` 可写 | 污染是否到达目标对象，是否跨请求保留 | 查找模板、配置、auth、path、child_process gadget |
| flat/deep merge 库绕过 | 循环引用、数组、点号路径或 CVE 版本是否匹配 | 固定库版本，构造最小 merge 样本 |
| Pug/EJS/模板选项被污染 | AST、filename、compileDebug、globals 等字段是否可控 | 转模板注入或命令执行链 |
| Node `vm` / vm2 / sandbox | 能否经 `constructor.constructor`、错误对象、dynamic import 触达宿主 | 按 CommonJS/ESM 分别测试 escape |
| Happy-DOM / SSR DOM | DOM API 是否会执行 script、`document.write` 或事件处理器 | 先确认服务端 DOM 与浏览器差异 |
| RSC / server action | Flight payload 或 server action 能否反序列化服务端对象 | 转 [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| 污染后需 SSRF/内部 require | 目标会从污染字段加载 URL、路径或模块 | 转 [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md) |

## 合并与拆分结论

- 保留为 family：原型污染、gadget 和 VM escape 共用 Node 对象模型，但落点不同，需要二级 pivot。
- 不合并进 `path-traversal-ssrf-upload-and-rsc.md`：后者负责资源定位链，本页负责 Node 对象/执行模型。
- 暂不拆 `prototype-pollution.md` 和 `node-vm-escape.md`：当前 raw 把污染、gadget 和 VM escape 串成链路，拆开会丢失转向关系。

## 常见误判

- 只证明污染成功，没有找到真实 gadget。
- 复用浏览器 JS 思路，忽略服务端 Node 版本、模块系统和沙箱暴露对象。
- payload 在单请求内有效，但目标应用每次重建对象，跨请求状态不保留。
- VM escape 成功后忘记回到题目链路，例如读取 secret、调用内部 API 或触发模板。

## 关联页面

- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [node-and-prototype.md](../raw/web/node-and-prototype.md)
