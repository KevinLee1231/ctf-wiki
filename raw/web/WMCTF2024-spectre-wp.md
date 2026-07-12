# Spectre

## 题目简述

题目是带 bot 的 Web 题。主站启用了 CSP nonce，同时设置了 `Cross-Origin-Opener-Policy: same-origin` 和 `Cross-Origin-Embedder-Policy: require-corp`，因此页面可使用 `SharedArrayBuffer`。flag 只能由 `admin` 访问，而伪造 admin token 需要 `token_key`。

源码显示 developer 访问分享页时会加载开发版模板：

```html
<script type="text/javascript" src="{{ script_src }}" nonce="{{ nonce }}" crossorigin></script>
<pre class="type-box code" data-lang="HTML"><code>{{ #if (role==="developer")}}{{ code }}{{ /if }}</code></pre>
```

`/assets/share-view.dev.js` 中会渲染出 `token_key` 的字节数组，并定义 `checker` 函数。

## 解题过程

### 关键机制

模板引擎存在二次渲染：

```js
content = content.replace(/{{ *#if *([\s\S]*?) *}}([\s\S]*?){{ *\/if *}}/g,
    (_, condition, block) => {
        if (Boolean(vm.runInNewContext(condition, data))) {
            return renderContentWithArgs(block, data);
        }
        return '';
    });
content = renderContentWithArgs(content, data);
```

当 `code` 位于 `if` 块内时，用户提交的 `{{ nonce }}` 会被第二次替换，从而构造带合法 nonce 的内联脚本，触发 XSS。

XSS 后有两种利用：

1. 非预期：污染原型或 hook `String.prototype.charCodeAt`，让 `checker("k")` 暴露调用栈或函数体中的 key。
2. 预期：使用 `SharedArrayBuffer` 高精度计时，观察 `checker(password, pos)` 逐字节比较的耗时差异，逐位恢复 `token_key`。

参考 URL：

- https://meltdownattack.com/
- https://www.yinchengli.com/2022/08/20/sharedarraybuffer-spectre/

### 求解步骤

提交 payload：

```html
<script nonce="{{ nonce }}">
document.addEventListener("DOMContentLoaded", () => {
  // leak token_key with checker side channel
});
</script>
```

`checker` 的逻辑可抽象为：

```js
function checker(password, pos_start = 0) {
    const token_key = [/* secret ascii */];
    return function () {
        for (let i = pos_start; i < 32; i++) {
            if (token_key[i] - password.charCodeAt(i) !== 0) {
                return false;
            }
        }
        return true;
    };
}
```

对每个位置枚举字符，多次计时并统计耗时最高的字符，即可恢复 key。拿到 `token_key` 后按 `src/token.mjs` 的签名方式伪造：

```js
const new_token = sign({ uid: account.uid, role: "admin" }, token_key);
fetch("/flag", { headers: { Cookie: `token=${new_token}` } });
```

## 方法总结

- XSS 根因：模板 `#if` 分支内二次渲染，用户内容可拿到 nonce。
- 泄密目标：developer 才能加载的 `share-view.dev.js` 内含 `token_key`。
- 外链关于 SharedArrayBuffer/Spectre 的关键信息是：COOP/COEP 允许高精度计时器，足以利用逐字节比较的时间差恢复秘密。
