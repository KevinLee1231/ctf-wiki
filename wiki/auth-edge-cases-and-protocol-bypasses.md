---
type: technique
tags: [web, technique]
skills: [ctf-web]
raw:
  - ../raw/web/auth-edge-cases-and-protocol-bypasses.md
updated: 2026-05-21
---

# Auth Edge Cases and Protocol Bypasses

## 适用场景

普通 cookie/JWT/hidden route 已不能解释现象，问题落在认证协议或数据结构边界：哈希桶碰撞、Unicode username 规范化、SRP/DH 公钥校验缺失、NoSQL/AQL 特殊语法导致权限提升。

## 识别信号

- 注册/登录允许大量可控用户名，且错误行为像哈希表退化或查找上限。
- 用户名经过 Unicode normalization、nodeprep、PRECIS、大小写折叠或同形字符处理。
- 认证协议出现 SRP、DH、group public value，且没有拒绝 0/N/小子群。
- 数据库查询不是 SQL，而是 AQL/NoSQL/ORM 表达式拼接。

## 最小证据

- 能构造两个不同输入在认证视角发生碰撞：同一 bucket、同一 normalized username、同一 session key 或同一 merged role。
- 协议题要能解释服务端计算出的 shared secret 为什么可预测。
- 注入题至少给出一个最小 payload，使返回用户或 role 发生变化。

## 解法骨架

1. 先排除普通 auth：cookie、JWT、hidden route、IDOR、默认口令。
2. Hash bucket：批量注册碰撞候选，观察目标用户查找是否退化或绕过。
3. Unicode：枚举同形字符/规范化形式，比较存储值和 lookup key。
4. SRP/DH：测试 `A=0`、`A=N`、`A=kN` 是否被拒绝；若不拒绝，session key 可预测。
5. AQL/NoSQL：定位字符串拼接点，用 `MERGE` / object merge 提升 role，再验证持久化与会话视角差异。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| `unordered_set` bucket collision | 攻击桶索引和探测上限，不需要完整 hash 碰撞。 |
| Unicode homograph | 写入和查询规范化不一致时，可伪装 admin。 |
| SRP `A=0` | shared secret 变成 0，攻击者知道 session key。 |
| AQL `MERGE` | 查询结果临时合并 admin role，绕过只看返回对象的 ACL。 |

## 常见陷阱

- 还没排除普通 auth 就跳长尾协议，增加复杂度。
- Unicode payload 被浏览器、代理或库自动规范化，测试结果失真。
- SRP/DH 只看密码强度，忽略公钥合法性校验。
- NoSQL payload 只复制 SQLi 思路，没有按目标查询语言语法构造。

## 关联技巧

- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-jwt.md](auth-jwt.md)
- [csp-xsleak-and-browser-exfiltration.md](csp-xsleak-and-browser-exfiltration.md)

## 原始资料

- [auth-edge-cases-and-protocol-bypasses.md](../raw/web/auth-edge-cases-and-protocol-bypasses.md)
