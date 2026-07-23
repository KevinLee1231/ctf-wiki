# Houston Astros' Sign Stealing Software - Part 2

## 题目简述

文件查看器把 POST 参数 `fileName` 直接交给 PHP 的 `file_get_contents`。代码虽然声明了路径穿越和扩展名正则，却从未真正执行这些检查，因此可以读取容器中的任意可读文本文件。

## 解题过程

先提交绝对路径 `/etc/passwd`，找到部署脚本创建的用户 `gitserver`。Dockerfile 把一个 Git 仓库的 `.git` 目录复制到：

```text
/home/gitserver/.git
```

继续通过 `fileName` 读取：

```text
/home/gitserver/.git/HEAD
/home/gitserver/.git/logs/HEAD
```

日志会给出提交对象与提交信息线索。部署 Dockerfile 对最新提交执行的 amend 信息为：

```text
VU1EQ1RGLXtZbzBfS04wd19UaDNfTjNYdF9wMXRDSH0=
```

Base64 解码后得到：

```text
UMDCTF-{Yo0_KN0w_Th3_N3Xt_p1tCH}
```

## 方法总结

声明过滤变量不等于实施过滤。任意文件读取后的下一步应从系统账户、应用配置和版本控制元数据建立路径；Git 的 HEAD、引用和日志可以泄露已经从工作树删除的信息。
