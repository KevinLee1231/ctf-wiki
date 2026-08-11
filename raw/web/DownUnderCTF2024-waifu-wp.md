# waifu

## 题目简述

WAIFU 在 `/auth` 前调用 LLM 检查原始 HTTP 请求；请求分析抛异常时，中间件只记录日志并继续放行。已登录管理员访问 `/auth/?redirectTo=...` 时，服务以 `window.location = "..."` 输出重定向目标，校验却只比较 `URL.hostname`，没有限制协议。攻击者可令题目机器人以管理员会话触发 `javascript:` URL，形成 XSS。

## 解题过程

第一层绕过来自错误处理。LLM 调用采用流式 API；对请求参数放入足够长的重复 URL 编码 token（官方示例重复 `%61`）可使模型端返回无效请求并抛异常。`waifuMiddleware` 的 `catch` 分支没有拒绝请求，直接执行 `next()`，因此恶意重定向参数到达后续逻辑。

第二层是重定向检查。程序以默认 `/flag/` 的 hostname 为基准，只比较：

```ts
redirectUrl.hostname === new URL(defaultRedirect).hostname
```

没有检查 `redirectUrl.protocol`。令目标以 `javascript://<题目应用主机>/` 开头，Node 的 `URL` 解析仍会把中间主机名视作与默认值一致；而浏览器最终执行的是 `javascript:` 后面的内容。HTML 实体编码会破坏直接嵌入的引号，因此将实际脚本放入 URL 编码数据或 Base64 参数中，在 `javascript:` 上下文内无引号地解码执行即可。

把这个 `/auth/...` 路径提交给题目的 bot 队列。机器人先通过只供 bot 使用的登录路由，获得管理员 session，再访问所给路径；XSS 可同源读取 `/flag/get` 的 JSON。只应在授权环境向自有接收端回传结果，题解不保留临时接收 URL。最终得到：

```text
DUCTF{t0kN_tOOooOOO0Kn_tooKN_t000000000Kn_x60_OwO_w0t_d15_n0_w4F?????questionmark???}
```

## 方法总结

安全网关在异常时必须默认拒绝，而不是 fail-open。重定向校验应采用允许协议与允许源的精确白名单，不能只比较 hostname；尤其要拒绝 `javascript:`、`data:` 等可执行方案。LLM 输出不能作为唯一的安全决策来源，即使模型可用也应由确定性的应用规则执行鉴权和输入校验。
