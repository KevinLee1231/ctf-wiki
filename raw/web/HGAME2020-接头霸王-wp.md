# 接头霸王

## 题目简述

服务会根据多个 HTTP 请求头逐步返回提示，要求请求来自指定站点、来自本机、使用指定浏览器标识，并满足一个与 2077 年更新时间有关的条件。修订后的题目还要求使用 `POST`。本题的核心是按服务端校验顺序构造请求头，而不是寻找传统 Web 漏洞。

## 解题过程

根据每次响应补齐对应条件：

1. `Referer: https://vidar.club/` 表示请求来源于指定站点。
2. `X-Forwarded-For: 127.0.0.1` 伪造反向代理记录的客户端地址，使其被识别为本地访问。
3. `User-Agent: Cosmos` 满足指定浏览器标识。
4. 响应中的 `Last-Modified` 指向 2077 年，因而补上同一时期的 `If-Unmodified-Since`。
5. 使用题目最终限定的 `POST` 方法发送请求。

完整请求骨架如下，比赛地址已经失效，因此用占位符表示：

```bash
curl -X POST '<challenge-url>/' \
  -H 'Referer: https://vidar.club/' \
  -H 'X-Forwarded-For: 127.0.0.1' \
  -H 'User-Agent: Cosmos' \
  -H 'If-Unmodified-Since: Sun, 03 Jan 2077 00:00:00 GMT'
```

`If-Unmodified-Since` 的标准语义是：只有资源自给定时间以来没有被修改，服务器才执行状态变更请求；否则应返回 `412 Precondition Failed`。题目把这个条件设计成了自定义校验，因此关键是结合响应中的 `Last-Modified` 构造日期，而不是依赖浏览器自动生成该请求头。

所有条件同时满足后，服务返回 flag。

## 方法总结

- 核心技巧：根据服务端逐步提示组合 `Referer`、代理来源、浏览器标识和条件请求头。
- 识别信号：响应内容反复描述“来源”“本地”“浏览器”“更新时间”，说明校验对象是 HTTP 方法和 Headers。
- 复用要点：代理来源头只能欺骗错误信任它的应用；真实环境中不能把 `X-Forwarded-For` 当作未经代理清洗的可信身份依据。
