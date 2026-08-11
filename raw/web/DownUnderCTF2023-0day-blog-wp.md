# DownUnderCTF 2023 0day Blog Writeup

## 题目简述

目标运行 Drupal 10.1.2，自定义模块把 `/flag` 路由限制为 `administrator` 角色。另一个服务提供管理员机器人：机器人先使用管理员账户登录 Drupal，再访问用户提交的 HTTP 或 HTTPS URL。

本题利用的不是 XSS，而是 Drupal 动态页面缓存对路由访问结果处理不完整。让管理员先访问 `/flag` 后，带 flag 的响应会进入一个没有按用户角色正确区分的缓存项；匿名请求随后可以命中该响应。

## 解题过程

题目路由定义如下：

```yaml
flag.content:
  path: '/flag'
  defaults:
    _controller: '\Drupal\flag\Controller\FlagController::index'
    _title: 'Get a flag!'
  requirements:
    _role: 'administrator'
```

控制器返回普通、可缓存的 render array，其中直接包含 flag。访问限制来自路由的 `_role` 检查，而不是控制器输出本身。

Drupal 的相关缺陷是：Dynamic Page Cache 在构造缓存变化条件时忽略了路由访问检查器返回的 cacheability metadata。即使 `_role` 的访问结果应按用户角色或权限变化，响应若没有从其他地方显式带上相同 cache context，动态页面缓存仍可能把它当成可跨用户复用的页面。Drupal 官方问题单 [#2973356](https://www.drupal.org/project/drupal/issues/2973356) 记录了这个根因；该行为后来在 Drupal 10.4.0 和 11.1.0 中修正。

利用步骤很短：

1. 在管理员机器人页面提交 Drupal 站点自身的 `/flag` 完整 URL，例如 `http://BLOG_HOST/flag`。
2. 机器人先登录为 `ghostccamm`，再携带管理员会话访问该 URL。路由检查通过，控制器返回含 flag 的页面，Dynamic Page Cache 保存该响应。
3. 等机器人访问完成后，以匿名会话再次请求相同的 `/flag`。动态缓存命中发生后，匿名用户拿到先前生成的管理员响应。

可用两个独立 Cookie Jar 复现“管理员预热、匿名读取”的边界；实际比赛中管理员会话由机器人维护，因此攻击者只需提交 URL 后再普通访问：

```bash
curl -X POST 'http://BOT_HOST/sendadminbot' \
  --data-urlencode 'url=http://BLOG_HOST/flag'

# 等待管理员机器人完成访问
curl 'http://BLOG_HOST/flag'
```

缓存页面中给出的 flag 为：

```text
DUCTF{c4cH3_m1_0uT51D3_w1Th_wHy_t0_n1T_c4Ch3_eHvrRiIet1nG!!1}
```

公开仓库中的控制器也保留了同一字符串，可用来核验结果，但无法替代对缓存越权机制的解释。

## 方法总结

访问控制不仅要在请求首次执行时正确，还必须参与缓存键和缓存失效条件。此题中 `_role` 本身没有配置错误；问题是路由访问结果的用户差异没有进入 Dynamic Page Cache 的变化维度。管理员先生成敏感响应后，缓存层便把它错误地复用给匿名用户。分析带管理员机器人的 CMS 题时，应同时检查页面内容、路由权限和缓存上下文，而不能只寻找脚本注入。
