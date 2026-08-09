# jujutsu-kaisen-2

## 题目简述

第二版修复了前一题的 Host 绕过：Nginx 固定向 Varnish 发送 `Host: jjk_app`。目标仍是读取内网 GraphQL 数据库中的角色备注。新的官方解法利用管理员浏览器、跨域带凭据上传、Varnish 的 ESI 解析和 PNG 完整性校验，构造逐字符 flag oracle。

## 解题过程

先在攻击者站点准备控制页、逐轮页面和 service worker，再把控制页 URL 交给访问机器人。机器人在同一无痕浏览上下文中登录目标后访问该站点，因而后续跨域 `fetch` 可以携带管理员会话。service worker 拦截本域的 `/interceptXX` 图片请求，把对应候选字符的 PNG 作为 multipart 文件，以 `mode: no-cors`、`credentials: include` 上传到目标 `/newchar`。

每张 PNG 的 `tEXt` 块中放入 ESI 指令，请求内网数据库：

```html
<esi:include src="http://jjk_db/?query=<notesLike-prefix-query>" onerror="continue" />
```

Varnish 对上传图片响应启用了 ESI 处理，所以它会代替浏览器访问 `jjk_db`，并把 GraphQL JSON 写回 PNG。查询条件测试 `notesLike: "%<已知前缀+候选字符>%"`。错误候选返回固定空结果：

```json
{"data":{"getCharacters":{"edges":[]}}}
```

官方脚本预先把 PNG 的 `tEXt` 长度和 CRC 修改成只对这段空结果有效。错误候选经 ESI 替换后仍是合法图片，触发 `onload`；正确候选返回非空 JSON，长度或 CRC 不再匹配，图片解码失败并触发 `onerror`。页面同时测试小写字母、数字、`}` 和 `{`，把唯一触发错误的字符发回控制端，然后注销旧 service worker 并开始下一轮。

从已知前缀 `maple{` 开始迭代，直到恢复右花括号，得到：

```text
maple{tooattachedforgottolookforunintendeds}
```

## 方法总结

这是一条不回显响应正文的组合侧信道：service worker 负责带凭据上传，Varnish ESI 负责服务器端请求内网，PNG 长度与 CRC 再把 GraphQL“空/非空”转换成浏览器的 load/error 事件。修补 Host 头只关闭了第一版捷径；对用户上传的非 HTML 内容启用 ESI 本身就危险，缓存层还应禁止访问内部管理后端，并对跨域带凭据请求使用严格的 SameSite、Origin 与 CSRF 校验。
