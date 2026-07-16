# findme2

## 题目简述

题面给出用户名 `WearyMeadow`，并说明对方把第二周 Misc WP 发布到博客、随后从 GitHub 删除了解锁密码。关键证据不在当前文件，而在公开仓库的提交历史：Git 删除一行并不会从旧提交中抹除它。

## 解题过程

在 GitHub 搜索用户后，可定位 [WearyMeadow 主页](https://github.com/WearyMeadow)。公开项目中，[wearymeadow.github.io](https://github.com/WearyMeadow/wearymeadow.github.io) 是博客仓库，另一个 AutoLoginBot 与题目主线无关：

![WearyMeadow 的 GitHub 主页显示博客仓库与公开活动，可用于确认题面用户名对应的账号](<0xGame2023-week2-findme2-wp/github-profile.jpg>)

博客中存在题为 `0xGame2023-Week2-Misc-WriteUp` 的加密文章：

![博客文章 0xGame2023-Week2-Misc-WriteUp 显示需要密码才能继续阅读](<0xGame2023-week2-findme2-wp/encrypted-post.jpg>)

进入仓库的 [提交历史](https://github.com/WearyMeadow/wearymeadow.github.io/commits/master/)，比较发布文章前后的提交；比赛时保留的 diff 截图显示，文章 front matter 中的 `password` 字段被删除，删除行本身仍会显示旧密码：

![GitHub 提交 diff 中 password 字段被删除，红色删除行保留了历史密码证据](<0xGame2023-week2-findme2-wp/password-removal-diff.jpg>)

若克隆仓库，可用以下只读命令搜索所有历史差异：

```bash
git log --all --oneline
git log --all -p -S 'password:'
git show "<commit>^:<path-to-post>"
```

把旧提交中的密码输入博客解锁框，文章正文给出本题 flag。仓库随题保存的 `Misc/Week2/findme2/1.txt` 也提供了离线校验值：

```text
0xGame{OHHHH_You_Find_Me_%%%}
```

公开网页和仓库历史可能在赛后改变，因此本地题目截图与 `1.txt` 是比赛时状态的证据快照，当前页面只用于核对账号、仓库和提交链路。

## 方法总结

OSINT 中“当前页面已删除”不等于历史证据消失。应依次确认账号归属、仓库用途、目标文章路径和引入/删除敏感字段的具体提交，并用提交 SHA 或本地截图固化证据；外链只作为原始来源，关键发现和最终结果必须写入正文。
