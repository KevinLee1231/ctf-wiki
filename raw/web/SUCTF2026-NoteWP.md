# SUCTF 2026 - SU_Note

## 题目简述

题目是一个用 PHP Session 保存用户与笔记的站点。容器启动时会生成随机管理员账号，并把 flag 作为管理员笔记写入会话；bot 每次接到任务都会创建一份管理员会话，将对应的 `PHPSESSID` 以 HttpOnly Cookie 写入浏览器，再访问用户提交的 URL。题面特意把 flag 主体限制为 `0/1`，说明预期解不是直接读取跨源响应，而是对管理员搜索结果做逐位侧信道探测。

官方解法利用 BFCache（Back/Forward Cache，往返缓存）容量侧信道。`search.php` 的“命中”和“未命中”分支会产生不同的 BFCache 资格，攻击页再用一串历史记录把这种差异放大为可观测信号。总 PDF 还给出了一条更直接的非预期解：`/bot/` 创建管理员会话时向外层响应泄露了新的 `PHPSESSID`，拿到它后可直接读取 flag。

## 解题过程

### 1. 源码中的敏感数据与搜索判定

`auth.php` 的 `initialize_system_admin()` 会在管理员名下维护一条固定笔记：

```php
$flagNote = [
    'id' => '__system_flag_note__',
    'title' => 'flag',
    'content' => system_flag_content(),
];
```

`search.php` 对当前会话中的笔记执行不区分大小写的子串搜索：

```php
return stripos($title, $searchQuery) !== false
    || stripos($content, $searchQuery) !== false;
```

因此，用管理员会话访问 `search.php?q=<已知前缀+候选位>` 时，“是否有结果”就是一个前缀 oracle。题目把未知部分约束为 `0/1`，每一位只需判断一个候选前缀即可。

### 2. 命中状态如何影响 BFCache

搜索页始终返回：

```php
header('Cache-Control: no-store');
```

这不再意味着页面一定不能进入 Chrome BFCache。Chrome 官方说明，现代 Chrome 会在满足安全条件时允许 `Cache-Control: no-store` 页面进入 BFCache，但保留时间更短，Cookie 或认证状态发生变化等情况仍会触发逐出。相关边界可见 [Chrome 的 `no-store` BFCache 说明](https://developer.chrome.com/docs/web-platform/bfcache-ccns?hl=zh-cn)。

真正区分两条分支的是页面末尾的 JavaScript：

```javascript
if (!isZeroResult) return;

const onUnload = () => {
    localStorage.setItem('su_eznote_last_miss_query', searchQuery);
};
window.addEventListener('unload', onUnload);
```

- 搜索命中：不注册 `unload`，搜索页有机会进入 BFCache；
- 搜索未命中：注册 `unload`，在题目使用的浏览器策略下会阻止该页进入 BFCache。

Chrome 文档也明确指出，桌面 Chrome 中的 `unload` 处理器与 BFCache 不兼容，并建议用 `pagehide` 或 `visibilitychange` 代替；见 [Chrome 的 `unload` 弃用说明](https://developer.chrome.com/docs/web-platform/deprecating-unload?hl=zh-cn)。这条利用依赖具体 Chrome 版本、策略与 BFCache 容量，复现时应使用题目 bot 环境，不能把本机浏览器结果直接当作定论。

### 3. 用历史记录链放大一页之差

官方 `exp.py` 在攻击者服务器上准备页面链：

```text
/a -> /b -> /c -> /d -> /e -> /f
   -> http://127.0.0.1:80/search.php?q=<candidate>
   -> /f -> /g -> history.back() -> ... -> /a
```

流程如下：

1. 把当前候选前缀和内网站点地址写入攻击页的 `sessionStorage`。
2. 让 bot 访问攻击服务器的 `/a`，依次导航到 `/f`。
3. `/f` 再把同一标签页导航到内网 `search.php`。管理员 Cookie 只发往 `127.0.0.1`，攻击者无需读取 Cookie。
4. `search.php` 等待约 5 秒后执行 `history.back()`，回到攻击者的 `/f`。
5. 随后进入 `/g` 并连续回退。每个攻击页通过 `pageshow.persisted` 判断自己是否从 BFCache 恢复，再用 `sendBeacon('/log?...')` 报告结果。

单标签页的 BFCache 槽位有限。搜索命中时，搜索页会额外占据一个槽位，较早的 `/a` 因而被淘汰；未命中时搜索页不能入缓存，`/a` 仍可保留。官方 exp 对最终信号的定义是：

```text
a_no_from_bfcache  -> hit
a_from_bfcache     -> miss
```

这里不要只数“页面数量”后凭直觉反推命中关系；应以题目浏览器中的实际 `pageshow.persisted` 结果和官方 exp 的信号映射为准。

### 4. 逐位恢复 flag

在公网可达的攻击机上启动官方 Flask exp，并让其内网站点参数保持为 `http://127.0.0.1:80`。对每一位执行：

1. 已知前缀初始设为实际 flag 前缀，例如 `SUCTF{`；
2. 先探测 `prefix + '0'`；
3. 让 bot 访问携带该候选值的攻击页 `/a`；
4. 收到 `hit` 就追加 `0`，收到 `miss` 就追加 `1`；
5. 重复直到恢复右花括号或达到题目给出的长度。

`/bot/` 对同一 IP 有“60 秒 3 次”的限速，且受最大并发数限制。脚本应等待明确的最终信号后再发下一次任务，并在超时、bot 失败或浏览器策略不匹配时重试，不能把暂时没有日志直接判为 `miss`。

### 5. 总 PDF 中的非预期会话泄露

总 PDF 使用了更短的链。先注册并登录普通用户，从页面取 CSRF token，然后向 `/bot/` 提交：

```text
action=visit
url=http://127.0.0.1:80/
```

关键漏洞在 `create_impersonated_session()`：

```php
$currentSessionId = session_id();
session_write_close();

$newSessionId = bin2hex(random_bytes(16));
session_id($newSessionId);
session_start();                 // 创建管理员会话
$_SESSION['auth_user'] = $username;
session_write_close();

session_id($currentSessionId);
session_start();                 // 恢复提交请求的用户会话
return $newSessionId;
```

服务器在同一个外层 HTTP 请求里切换并启动两份 PHP Session，因而可能在 `/bot/` 的 302 响应中留下多个 `Set-Cookie: PHPSESSID=...`。提交请求时禁止自动跟随重定向，收集所有 `Set-Cookie`，排除自己的会话值，剩下的就是新建的 bot/admin 会话。

这不是 `bot.js` 把目标网页的响应头代理回来：源码中的 bot 只用 Puppeteer `page.goto()`，最后输出 `BOT_OK`，没有转发目标响应。真正泄露的是 PHP 服务端切换会话时生成的外层响应头。拿到泄露值后直接请求：

```http
GET /search.php?q=SUCTF HTTP/1.1
Cookie: PHPSESSID=<leaked_session>
```

搜索结果会显示管理员 flag 笔记；也可访问 `/` 读取默认选中的第一条笔记。这条非预期解绕过了整个 BFCache 逐位侧信道，但是否存在取决于实际部署的 PHP Session 响应头行为，不能代替对预期机制的理解。

## 方法总结

- 预期技巧是 BFCache XS-Leak：通过“命中时可缓存、未命中时因 `unload` 不可缓存”的差异，结合有限缓存槽位和 `pageshow.persisted` 构造布尔 oracle。
- 跨源限制并不妨碍攻击：攻击页只负责导航管理员标签页并观察自己页面的恢复状态，不需要读取内网响应。
- 浏览器侧信道高度依赖版本、策略、缓存容量与时序；利用脚本必须区分 `hit`、`miss` 和“无有效信号”。
- 非预期解来自服务端 Session 生命周期错误。审计 bot 功能时，除了 XSS、SSRF 和 Cookie 作用域，还要检查服务端为 bot 创建身份时是否把新会话标识写进了攻击者可见的响应。
