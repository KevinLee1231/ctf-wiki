# Wiki

## 题目简述

本题延续 `Horoscopes` 的 Gemini capsule。入口页面给出一个包含大量相对链接的 Wiki 索引，目标是在这些 Gemtext 页面中找到以 `DUCTF{` 开头的内容。核心工作是解析 Gemini 响应和 `=>` 链接并做有界遍历，而不是猜测页面名。

## 解题过程

可以用 Gemini 客户端逐页搜索，也可以写一个小型 crawler。遍历器需要完成四件事：

1. 从题目 Gemini 根 URL 开始请求页面。
2. 只处理状态码 `20` 的成功响应。
3. 当响应元信息以 `text/gemini` 开头时，解析每个 `=> <url> [label]` 链接。
4. 把相对链接按当前 URL 解析为绝对 URL，并用 `visited` 集合去重。

官方 Go solver 的核心逻辑可精简为：

```go
resp, err := client.Fetch(currentURL)
if err != nil || resp.Status != 20 {
    return
}

body, _ := io.ReadAll(resp.Body)
if strings.Contains(string(body), "DUCTF{") {
    fmt.Println(currentURL, string(body))
}

if strings.HasPrefix(resp.Meta, "text/gemini") {
    for _, line := range strings.Split(string(body), "\n") {
        if strings.HasPrefix(line, "=>") {
            fields := strings.Fields(line)
            next, err := baseURL.Parse(fields[1])
            if err == nil && !visited[next.String()] {
                crawl(next.String())
            }
        }
    }
}
```

索引中的 `pages/rabid_bean_potato.gmi` 命中搜索，正文末尾给出：

```text
DUCTF{rabbit_is_rabbit_bean_is_bean_potato_is_potato_banana_is_banana_carrot_is_carrot}
```

## 方法总结

- 核心技巧：按 Gemtext 语法递归解析相对链接，并对页面正文做 flag 前缀搜索。
- 识别信号：题面说明是 `Horoscopes` 续题，页面被描述为 Wiki，入口又包含大量 `=>` 链接。
- 复用要点：crawler 必须规范化 URL 并去重，限制在题目 capsule 的同源范围；只对成功的 `text/gemini` 响应继续抽取链接，避免把错误页或二进制响应当文本递归。
