---
type: technique
tags: [web, auth, session, access-control, state-confusion]
skills: [ctf-web]
raw:
  - ../raw/web/auth-bypass-cookies-and-hidden-routes.md
  - ../raw/web/auth-jwt.md
updated: 2026-07-27
---

# Session and Access-Control State Confusion

## 适用场景

认证、授权、代理 ACL 和业务状态分别读取不同 cookie、header、path 或 session 字段，攻击者可制造“已登录但未授权”“代理允许但后端拒绝”等视图差异。

## 识别信号

- 同一请求经代理、框架和业务层后得到不同身份。
- 多个 cookie/header 表达用户、角色、租户或登录状态。
- 隐藏路由、方法切换、路径规范化或 session fixation 改变访问结果。

## 最小证据

- 对匿名、普通用户和目标角色建立请求差分。
- 明确每一层读取的身份字段和规范化后的 path/method。
- 证明绕过的是授权边界，而不只是访问公开别名。

## 解法骨架

1. 记录完整登录流程、cookie 生命周期和重定向。
2. 单变量修改 cookie、header、path、method 与重复参数。
3. 比较代理日志/响应与后端业务状态，定位谁信任了哪一份身份。
4. 构造最小状态组合并验证跨用户、跨租户或管理员资源访问。

## 关键变体

- Signed/unsigned state 混用。
- Proxy-backend 路径/方法视图差异。
- Session fixation、角色字段或多 cookie 优先级冲突。

## 常见陷阱

- 只看 HTTP 200，没有验证资源所有者和权限提升。
- 浏览器自动合并 cookie，掩盖重复字段顺序。
- 登录重定向产生新 session，却继续测试旧 token。

## 关联技巧

- [auth-bypass-cookies-and-hidden-routes.md](auth-bypass-cookies-and-hidden-routes.md)
- [auth-jwt.md](auth-jwt.md)
- [request-view-normalization-differentials.md](request-view-normalization-differentials.md)

## 原始资料

- [auth-bypass-cookies-and-hidden-routes.md](../raw/web/auth-bypass-cookies-and-hidden-routes.md)
- [auth-jwt.md](../raw/web/auth-jwt.md)
