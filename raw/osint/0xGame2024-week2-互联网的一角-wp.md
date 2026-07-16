# 互联网的一角

## 题目简述

题目提供了一个网页。页面本身没有直接给出 flag，需要从前端注释、域名解析结果和 GitHub Pages 的公开历史中还原曾经存在的内容。

## 解题过程

查看网页源代码，可以找到注释：

```html
<!--这个网址好像指向了另一个网址......-->
```

这说明入口域名 `www.challenge.st4rt.top` 只是别名，应继续查询其 DNS 记录。使用 `nslookup`、`dig` 或能够显示规范名称的 `ping` 均可；查询结果指向：

```text
oxg4me2024.github.io
```

`github.io` 是 GitHub Pages 的托管域名，其中子域名通常对应 GitHub 用户名。因此转到用户 `Oxg4me2024` 的公开仓库，可以找到 `CTF_Challenge`，并在 `f14g_1s_h3re/flag.txt` 中发现线索。当前版本的文件只有：

```text
Hey, where is my flag???
```

这表明 flag 已经被后续提交覆盖。打开文件历史并回溯最早的创建提交即可恢复原内容。仓库的 [原始提交 e4ee501](https://github.com/Oxg4me2024/CTF_Challenge/commit/e4ee501) 显示，该提交首次创建 `flag.txt`，其中写入了真正的 flag。

```text
0xGame{W0w_u_kn0w_g1thub_p4ges!!}
```

## 方法总结

本题的线索链是“HTML 注释 → CNAME → GitHub Pages 用户 → 仓库文件 → Git 提交历史”。遇到公开页面中的内容被删除或覆盖时，不应只查看当前文件，还要检查 Git 历史、具体提交和差异记录；稳定的提交链接也比只链接仓库首页更便于复核。
