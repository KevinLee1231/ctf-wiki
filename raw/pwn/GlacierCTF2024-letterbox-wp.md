# GlacierCTF 2024 LetterBox

## 题目简述

题目同时提供一个 C 语言信件客户端和 Flask 管理员收件箱。客户端只允许信件正文包含字母数字，发送后 Web 端却把信件直接拼进模板并调用 `render_template_string()`；如果能绕过字符检查注入 `{{...}}`，即可触发 Jinja2 SSTI，并从 Flask config 读取 flag。

绕过点是信件对象的 use-after-free。删除信件后，全局指针没有清空；随后注册用户会复用同一个 malloc chunk，使悬空的 `Letter.content` 实际指向不受字母数字限制的 `User.username`。

## 解题过程

### 1. 确认 Web 端 SSTI 汇点

Socket 服务收到信件后逐行追加到 `letters.txt`。管理员访问 `/inbox` 时，程序先把最近 10 封信拼入 HTML，再执行：

```python
return render_template_string(html)
```

应用启动时还执行：

```python
app.config['FLAG'] = open('./flag.txt').read()
```

所以 `{{config}}` 就足以把包含 `FLAG` 的配置字典渲染进页面。问题只在于 `create_letter()` 会拒绝 `{`：

```c
if (!is_alphanumeric(content)) {
    puts("Invalid characters in letter content!");
    return;
}
```

### 2. 让 User 复用已释放 Letter

两个 packed 结构为：

```c
struct User {
    char username[128];
    char password[128];
};

struct Letter {
    char content[256];
    size_t id;
};
```

请求大小分别是 256 和 264 字节，但在 64 位 glibc 中向上对齐后都进入同一个 0x110 大小的 chunk。删除逻辑只调用：

```c
free(letters[id]);
```

没有执行 `letters[id] = NULL`，`letter_count` 也不减少。原 `id` 位于该 chunk 的最后 8 字节，注册新用户只覆盖前 256 字节，因此旧 ID 通常仍保留，发送函数也不会再次验证对象类型。

按以下顺序操作：

1. 注册并登录普通用户 `a/a`；
2. 创建内容为 `a` 的信件 0；
3. 删除信件 0，使其进入 tcache；
4. 退出登录，注册用户名和密码均为 `{{config}}` 的新用户；
5. 新 `User` 复用旧信件 chunk，悬空的 `letters[0]->content` 现在从用户名开头读取；
6. 登录任一已注册账户并选择 `Send all letters`。

用户名输入路径没有 `is_alphanumeric()`，所以花括号可以进入堆块。`strlen(letters[0]->content)` 在用户名末尾的 NUL 停止，Socket 收到的正好是 `{{config}}`。

### 3. 登录收件箱触发模板求值

Web 登录逻辑每次生成字符串 `admin` 的新哈希，再用用户提交密码校验，因此管理员密码固定是：

```text
admin
```

登录后访问 `/inbox`，Jinja2 对信件再次解释，页面中出现 Flask 配置及：

```text
gctf{U4F_7UrN3d_1N7O_5571}
```

## 方法总结

本题由两个不同边界的漏洞串联：“C 客户端 UAF/类型混淆 → 绕过信件字符集 → Flask `render_template_string` SSTI”。如果只看到 SSTI 而忽略字母数字过滤，链条无法落地；只利用 UAF 也不会直接执行代码。修复应在 free 后清空指针并维护对象生命周期，发送时验证有效对象；Web 端则应把信件作为模板变量转义输出，绝不能把用户内容拼入待解析模板。
